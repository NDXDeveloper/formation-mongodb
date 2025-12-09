🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 7 : Développement et Intégration (Intermédiaire à Avancé)

## 🎯 De la base de données à l'application

Vous maîtrisez maintenant MongoDB en profondeur : modélisation, performance, architecture distribuée, sécurité et cloud. Vous savez créer des clusters résilients et optimisés. Mais toute cette expertise n'a de valeur que si elle est **correctement intégrée dans vos applications**. Comment connecter efficacement votre code à MongoDB ? Quels patterns utiliser ? Comment gérer les erreurs, les connexions, les transactions dans un contexte applicatif ?

La Partie 7 est dédiée au **développement d'applications avec MongoDB**, couvrant l'intégration dans différents langages, les drivers officiels, les ODM/ORM, et les fonctionnalités avancées qui font de MongoDB une base de données parfaite pour les applications modernes.

## 💻 L'intégration applicative : Le pont entre la base et le métier

### De la requête MongoDB au code applicatif

**Le parcours d'une requête :**

```
Application Code
     ↓
Driver MongoDB (Node.js, Python, Java, etc.)
     ↓
Connection Pool
     ↓
Network (TCP/TLS)
     ↓
mongos (si sharding) ou mongod
     ↓
Query Engine
     ↓
Storage Engine (WiredTiger)
     ↓
Disque
```

Chaque étape présente des défis et des opportunités d'optimisation. Une intégration mal faite peut ruiner les performances d'une base parfaitement configurée.

### Les niveaux d'abstraction

**Niveau 1 : Driver brut (Low-level)**
```javascript
// Node.js avec driver natif
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
await client.connect();
const db = client.db('shop');
const result = await db.collection('products').findOne({ _id: productId });
```

**Avantages :**
- ✅ Contrôle total
- ✅ Performance optimale
- ✅ Accès à toutes les fonctionnalités

**Inconvénients :**
- ❌ Code verbeux
- ❌ Pas de validation automatique
- ❌ Mapping manuel vers objets

---

**Niveau 2 : ODM (Object-Document Mapper)**
```javascript
// Mongoose (Node.js ODM)
const productSchema = new Schema({
  name: { type: String, required: true },
  price: { type: Number, min: 0 },
  category: String
});
const Product = mongoose.model('Product', productSchema);
const product = await Product.findById(productId);
```

**Avantages :**
- ✅ Code concis et expressif
- ✅ Validation automatique
- ✅ Mapping automatique vers objets
- ✅ Middlewares et hooks
- ✅ Abstraction des détails

**Inconvénients :**
- ❌ Overhead de performance (léger)
- ❌ Courbe d'apprentissage
- ❌ Parfois trop "magique"
- ❌ Peut masquer ce qui se passe réellement

---

**Niveau 3 : Query Builders et ORMs**
```javascript
// Prisma (ORM moderne)
const product = await prisma.product.findUnique({
  where: { id: productId }
});
```

**Avantages :**
- ✅ Type-safety (TypeScript)
- ✅ Auto-completion excellente
- ✅ Migrations automatiques
- ✅ Multi-database (pas que MongoDB)

**Inconvénients :**
- ❌ Abstraction très élevée
- ❌ Moins de contrôle
- ❌ Peut ne pas supporter toutes les fonctionnalités MongoDB

---

**Choix selon le contexte :**

```
Driver natif : Microservices performants, contrôle fin
ODM (Mongoose, etc.) : Applications traditionnelles, équipes orientées OOP
Query Builder/ORM : Applications TypeScript, équipes polyglot (multi-DB)
```

**Recommandation générale :** Commencez avec un ODM pour la productivité, passez au driver natif si la performance devient critique.

### Patterns d'intégration modernes

**1. Connection Pooling : Ne jamais créer une connexion par requête**

❌ **Anti-pattern :**
```javascript
async function getUser(id) {
  const client = new MongoClient(url);  // ❌ Nouvelle connexion !
  await client.connect();
  const user = await client.db().collection('users').findOne({ _id: id });
  await client.close();  // ❌ Fermeture immédiate !
  return user;
}
```

**Problème :**
- Latence énorme (handshake TCP + TLS à chaque requête)
- Épuisement des connexions disponibles
- Scalabilité catastrophique

✅ **Bonne pratique :**
```javascript
// Connection unique réutilisée (pool interne)
const client = new MongoClient(url, {
  maxPoolSize: 50,
  minPoolSize: 10
});
await client.connect();  // Une seule fois au démarrage

async function getUser(id) {
  // Réutilise une connexion du pool
  return await client.db().collection('users').findOne({ _id: id });
}
```

**Bénéfices :**
- Latence réduite de 100-500ms à 1-5ms
- Scalabilité linéaire
- Gestion automatique des connexions

---

**2. Repository Pattern : Séparer la logique de données**

```javascript
// repositories/UserRepository.js
class UserRepository {
  constructor(db) {
    this.users = db.collection('users');
  }

  async findById(id) {
    return await this.users.findOne({ _id: new ObjectId(id) });
  }

  async findByEmail(email) {
    return await this.users.findOne({ email });
  }

  async create(userData) {
    const result = await this.users.insertOne(userData);
    return { _id: result.insertedId, ...userData };
  }

  async update(id, updates) {
    return await this.users.updateOne(
      { _id: new ObjectId(id) },
      { $set: updates }
    );
  }
}

// services/UserService.js
class UserService {
  constructor(userRepository) {
    this.userRepository = userRepository;
  }

  async register(email, password) {
    // Logique métier
    const hashedPassword = await bcrypt.hash(password, 10);
    return await this.userRepository.create({
      email,
      password: hashedPassword,
      createdAt: new Date()
    });
  }
}
```

**Avantages :**
- ✅ Séparation des responsabilités
- ✅ Testabilité (mock facile)
- ✅ Réutilisabilité
- ✅ Maintenabilité

---

**3. Unit of Work : Gestion transactionnelle cohérente**

```javascript
class UnitOfWork {
  constructor(client) {
    this.session = client.startSession();
    this.repositories = {
      users: new UserRepository(client.db(), this.session),
      orders: new OrderRepository(client.db(), this.session)
    };
  }

  async execute(work) {
    this.session.startTransaction();
    try {
      const result = await work(this.repositories);
      await this.session.commitTransaction();
      return result;
    } catch (error) {
      await this.session.abortTransaction();
      throw error;
    } finally {
      await this.session.endSession();
    }
  }
}

// Usage
const uow = new UnitOfWork(client);
await uow.execute(async ({ users, orders }) => {
  const user = await users.findById(userId);
  user.balance -= orderTotal;
  await users.update(userId, { balance: user.balance });
  await orders.create({ userId, total: orderTotal, items });
});
```

---

**4. Event-Driven avec Change Streams**

```javascript
// Event publisher
const changeStream = db.collection('orders').watch();

changeStream.on('change', async (change) => {
  if (change.operationType === 'insert') {
    // Publier événement
    await eventBus.publish('order.created', change.fullDocument);
  }
});

// Event subscriber
eventBus.subscribe('order.created', async (order) => {
  await sendOrderConfirmationEmail(order);
  await updateInventory(order);
  await notifyShipping(order);
});
```

**Use cases :**
- Microservices communication
- Real-time notifications
- Audit logging
- Data replication entre systèmes

---

**5. Caching Strategy : Réduire la charge sur MongoDB**

```javascript
class CachedRepository {
  constructor(repository, cache) {
    this.repository = repository;
    this.cache = cache;  // Redis, Memcached, etc.
  }

  async findById(id) {
    // Cache-aside pattern
    const cached = await this.cache.get(`user:${id}`);
    if (cached) return JSON.parse(cached);

    const user = await this.repository.findById(id);
    if (user) {
      await this.cache.setex(`user:${id}`, 3600, JSON.stringify(user));
    }
    return user;
  }

  async update(id, updates) {
    await this.repository.update(id, updates);
    await this.cache.del(`user:${id}`);  // Invalidation
  }
}
```

**Stratégies de cache :**
- **Cache-aside** : App gère le cache
- **Read-through** : Cache gère le load depuis DB
- **Write-through** : Écriture dans cache + DB simultanément
- **Write-behind** : Écriture async vers DB

---

**6. Microservices Pattern : Database per Service**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  User Service   │     │  Order Service  │     │ Product Service │
│                 │     │                 │     │                 │
│  ┌───────────┐  │     │  ┌───────────┐  │     │  ┌───────────┐  │
│  │MongoDB    │  │     │  │MongoDB    │  │     │  │MongoDB    │  │
│  │users DB   │  │     │  │orders DB  │  │     │  │products DB│  │
│  └───────────┘  │     │  └───────────┘  │     │  └───────────┘  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         ↓                       ↓                       ↓
    ┌────────────────────────────────────────────────────────┐
    │              Event Bus (Kafka, RabbitMQ)               │
    └────────────────────────────────────────────────────────┘
```

**Principe :** Chaque service possède sa propre base MongoDB, pas de base partagée.

**Communication :**
- Events (async) : Preferred pour eventual consistency
- API calls (sync) : Pour cohérence forte nécessaire

**Challenges :**
- Transactions distribuées (saga pattern)
- Data consistency (eventual)
- Data duplication (denormalization between services)

## 📋 Prérequis

Cette partie s'adresse à des **développeurs** ayant :

### Connaissances MongoDB requises
- ✅ **Maîtrise des Parties 1-2** (fondamentaux, modélisation, requêtes)
- ✅ Compréhension de la Partie 3 (transactions) - utile mais pas critique
- ✅ Connaissance des index et optimisations
- ✅ Expérience avec mongosh et Compass

### Compétences en développement
- ✅ **Maîtrise d'au moins un langage** : JavaScript/Node.js, Python, Java, C#, Go, PHP, ou Ruby
- ✅ **Programmation orientée objet** : Classes, héritage, polymorphisme
- ✅ **Programmation asynchrone** : Promises, async/await, callbacks
- ✅ **APIs REST** : Création et consommation
- ✅ **Gestion d'erreurs** : Try/catch, error handling patterns
- ✅ **Tests** : Unit tests, integration tests (notions)

### Expérience applicative
- 💼 Développement d'au moins une application complète
- 💼 Intégration d'une base de données (SQL ou NoSQL)
- 💼 Compréhension des patterns architecturaux (MVC, layered, etc.)
- 💼 Expérience avec un framework web (Express, Django, Spring, etc.)

### Outils et environnement
- 🛠️ Git et versioning
- 🛠️ npm/pip/maven (gestionnaires de packages)
- 🛠️ IDE ou éditeur avec autocompletion
- 🛠️ Postman ou équivalent pour tester APIs

### État d'esprit
- 🧠 Focus sur la qualité du code
- 🧠 Refactoring continu
- 🧠 Tests comme partie intégrante du développement
- 🧠 Documentation du code
- 🧠 Curiosité pour les bonnes pratiques

**Note** : Cette partie est accessible aux développeurs intermédiaires. Les exemples couvriront principalement Node.js/JavaScript pour la cohérence, mais les concepts s'appliquent à tous les langages.

## 🎓 Objectifs d'apprentissage

À la fin de cette partie, vous serez capable de :

### Compétences d'intégration

**Connection et configuration :**
- ✅ **Connecter** une application à MongoDB avec le driver approprié
- ✅ **Configurer** le connection pooling correctement
- ✅ **Gérer** les connection strings (local, Atlas, replica sets)
- ✅ **Comprendre** les options de connexion et leur impact
- ✅ **Implémenter** la retry logic pour la résilience
- ✅ **Gérer** les erreurs de connexion gracefully

**Drivers officiels :**
- ✅ **Maîtriser** le driver de votre langage principal
- ✅ **Comprendre** les différences entre drivers
- ✅ **Choisir** entre driver natif et ODM/ORM
- ✅ **Utiliser** les fonctionnalités avancées des drivers

**ODM/ORM :**
- ✅ **Utiliser** Mongoose (Node.js) ou équivalent
- ✅ **Définir** des schémas et modèles
- ✅ **Implémenter** la validation
- ✅ **Utiliser** les middlewares et hooks
- ✅ **Comprendre** les compromis performance

### Compétences en patterns et architecture

**Patterns applicatifs :**
- ✅ **Implémenter** le Repository Pattern
- ✅ **Utiliser** le Unit of Work pour les transactions
- ✅ **Appliquer** le Service Layer Pattern
- ✅ **Gérer** les dépendances (Dependency Injection)
- ✅ **Structurer** le code en couches (layered architecture)

**Gestion des erreurs :**
- ✅ **Gérer** les erreurs MongoDB spécifiques
- ✅ **Implémenter** le retry logic intelligent
- ✅ **Logger** les erreurs efficacement
- ✅ **Communiquer** les erreurs aux clients (HTTP status codes, etc.)

**Performance :**
- ✅ **Optimiser** les requêtes dans le code
- ✅ **Utiliser** les indexes intelligemment
- ✅ **Implémenter** le caching
- ✅ **Éviter** les N+1 queries
- ✅ **Batching** et bulkWrite

### Compétences en fonctionnalités avancées

**Change Streams :**
- ✅ **Écouter** les changements de données en temps réel
- ✅ **Filtrer** les événements pertinents
- ✅ **Implémenter** des workflows event-driven
- ✅ **Gérer** les resume tokens pour la résilience

**GridFS :**
- ✅ **Stocker** des fichiers volumineux (> 16 MB)
- ✅ **Stream** des fichiers efficacement
- ✅ **Gérer** les métadonnées
- ✅ **Choisir** entre GridFS et stockage externe (S3)

**Time Series Collections :**
- ✅ **Modéliser** des données temporelles (IoT, metrics)
- ✅ **Optimiser** les requêtes de séries temporelles
- ✅ **Comprendre** les avantages vs collections standard

**Géospatial :**
- ✅ **Stocker** des coordonnées géographiques
- ✅ **Effectuer** des requêtes géospatiales ($near, $geoWithin)
- ✅ **Créer** des index géospatiaux (2dsphere)
- ✅ **Implémenter** des features comme "trouver les restaurants à moins de 5km"

**Full-Text Search :**
- ✅ **Créer** des index texte
- ✅ **Effectuer** des recherches full-text avancées
- ✅ **Comprendre** quand utiliser $text vs Atlas Search

### Compétences transversales

**Tests :**
- ✅ **Écrire** des unit tests avec MongoDB en mock
- ✅ **Écrire** des integration tests avec MongoDB réel
- ✅ **Utiliser** MongoDB Memory Server pour les tests
- ✅ **Tester** les edge cases (erreurs réseau, timeouts, etc.)

**Sécurité applicative :**
- ✅ **Prévenir** les injections NoSQL
- ✅ **Valider** les entrées utilisateur
- ✅ **Ne jamais** exposer les erreurs MongoDB au client
- ✅ **Utiliser** des variables d'environnement pour les credentials

**Documentation :**
- ✅ **Documenter** les schémas de données
- ✅ **Commenter** le code complexe
- ✅ **Générer** la documentation API
- ✅ **Maintenir** un changelog

## 📚 Vue d'ensemble des modules

Cette partie contient **2 modules complémentaires** :

### Module 15 : Drivers et Intégration Applicative
**Durée estimée : 18-22 heures**

Le cœur de l'intégration applicative : drivers, patterns et bonnes pratiques.

#### 15.1 Vue d'ensemble des drivers officiels
**Durée : 2 heures**

Comprendre l'écosystème des drivers MongoDB.

**Ce que vous maîtriserez :**
- Les drivers officiels disponibles (10+ langages)
- Architecture des drivers (CRUD, connection pooling, etc.)
- Versioning et compatibility
- Communauté et support

**Drivers principaux :**
- Node.js (JavaScript/TypeScript)
- Python (PyMongo)
- Java
- C# / .NET
- Go
- PHP
- Ruby
- Rust, C++, Scala, etc.

---

#### 15.2-15.8 Drivers par langage
**Durée : 8-10 heures**

Deep dive dans les drivers majeurs.

**Node.js / JavaScript :**
- Driver MongoDB natif
- Async/await et Promises
- Connection pooling
- Error handling
- TypeScript support

**Python (PyMongo) :**
- Installation et setup
- CRUD operations
- Context managers
- Motor (async pour asyncio)
- Type hints

**Java :**
- Driver sync vs async
- POJO mapping
- Spring Data MongoDB
- Reactive streams

**C# / .NET :**
- MongoDB.Driver
- LINQ queries
- Async/await pattern
- ASP.NET Core integration

**Go :**
- mongo-go-driver
- Context management
- Struct mapping
- Concurrency patterns

**Autres langages :**
- PHP (avec MongoDB extension)
- Ruby (Mongoid ODM)

**Focus principal :** Le module couvrira en détail 2-3 langages (Node.js, Python, Java) avec des aperçus des autres.

---

#### 15.9 Connection String et options
**Durée : 2-3 heures**

Configuration avancée des connexions.

**Ce que vous maîtriserez :**

**Format de connection string :**
```
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/[defaultauthdb][?options]]
```

**Options critiques :**
- `maxPoolSize` : Taille max du pool (défaut 100)
- `minPoolSize` : Taille min du pool
- `maxIdleTimeMS` : Timeout pour connexions inactives
- `serverSelectionTimeoutMS` : Timeout pour sélection de serveur
- `retryWrites` : Retry automatique des écritures (défaut true)
- `retryReads` : Retry automatique des lectures
- `w` : Write concern
- `readPreference` : Préférence de lecture

**Exemples :**
```javascript
// Développement local
mongodb://localhost:27017/mydb

// Atlas avec options
mongodb+srv://user:pass@cluster.mongodb.net/mydb?retryWrites=true&w=majority

// Replica Set
mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=myRS

// Avec toutes les options
mongodb://user:pass@host:27017/db?maxPoolSize=50&w=majority&readPreference=primaryPreferred
```

---

#### 15.10 Connection Pooling
**Durée : 2-3 heures**

Gestion efficace des connexions.

**Ce que vous maîtriserez :**
- Fonctionnement du connection pool
- Dimensionnement du pool
- Monitoring du pool
- Troubleshooting (exhaustion, leaks)

**Best practices :**
- 1 MongoClient par application (singleton)
- Pool size = (nombre de threads/workers) × 2
- Monitoring des connexions actives
- Graceful shutdown

**Métriques à surveiller :**
```javascript
// Node.js driver monitoring
client.on('connectionPoolCreated', (event) => {
  console.log('Pool created', event);
});

client.on('connectionCheckedOut', (event) => {
  // Connexion prise du pool
});

client.on('connectionCheckedIn', (event) => {
  // Connexion rendue au pool
});
```

---

#### 15.11 Gestion des erreurs et retry
**Durée : 2-3 heures**

Résilience applicative.

**Types d'erreurs MongoDB :**
- **Erreurs réseau** : Connection timeout, network error
- **Erreurs de données** : Duplicate key, validation failed
- **Erreurs de ressources** : Out of memory, disk full
- **Erreurs transactionnelles** : Transaction aborted

**Retry strategies :**
```javascript
async function withRetry(operation, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } catch (error) {
      if (!isRetriableError(error) || i === maxRetries - 1) {
        throw error;
      }
      // Exponential backoff
      await sleep(Math.pow(2, i) * 100);
    }
  }
}

function isRetriableError(error) {
  // Network errors, transaction aborts, etc.
  return error.hasErrorLabel('RetryableWriteError') ||
         error.hasErrorLabel('TransientTransactionError');
}
```

---

#### 15.12 ODM et ORM
**Durée : 2-3 heures**

Abstraction orientée objet.

**Overview des solutions :**

**JavaScript/Node.js :**
- **Mongoose** : Le plus populaire, feature-rich
- **Prisma** : Type-safe, multi-DB
- **TypeORM** : Support MongoDB + SQL

**Python :**
- **MongoEngine** : ODM Django-like
- **Beanie** : ODM pour FastAPI (async)

**Java :**
- **Spring Data MongoDB** : Intégration Spring
- **Morphia** : ODM type-safe

**C# :**
- **MongoDB.Entities** : Strongly-typed

---

#### 15.12.1 Mongoose (Node.js)
**Durée : 3-4 heures**

Deep dive dans Mongoose.

**Ce que vous maîtriserez :**
- Définition de schémas
- Types et validation
- Virtuals et methods
- Middlewares (pre/post hooks)
- Population (équivalent JOIN)
- Plugins

**Exemple complet :**
```javascript
const userSchema = new Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    validate: {
      validator: (v) => /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/.test(v),
      message: 'Invalid email'
    }
  },
  password: {
    type: String,
    required: true,
    minlength: 8
  },
  profile: {
    firstName: String,
    lastName: String,
    age: { type: Number, min: 0, max: 120 }
  },
  createdAt: { type: Date, default: Date.now }
});

// Virtual (ne sont pas stockés)
userSchema.virtual('fullName').get(function() {
  return `${this.profile.firstName} ${this.profile.lastName}`;
});

// Middleware pre-save
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Method instance
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

// Static method
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email });
};

const User = mongoose.model('User', userSchema);
```

---

#### 15.12.2-15.12.3 Autres ODM/ORM
**Durée : 2 heures**

Aperçu des alternatives.

**Motor (Python async) :**
```python
import motor.motor_asyncio

client = motor.motor_asyncio.AsyncIOMotorClient('mongodb://localhost:27017')
db = client.test_database
collection = db.test_collection

async def do_insert():
    document = {'key': 'value'}
    result = await collection.insert_one(document)
```

**Spring Data MongoDB :**
```java
@Document(collection = "users")
public class User {
    @Id
    private String id;

    @Indexed(unique = true)
    private String email;

    private String name;
}

public interface UserRepository extends MongoRepository<User, String> {
    User findByEmail(String email);
    List<User> findByNameContaining(String name);
}
```

---

#### 15.13 Bonnes pratiques d'intégration
**Durée : 2-3 heures**

Synthèse des best practices.

**Checklist :**
- [ ] Utiliser connection pooling
- [ ] Implémenter retry logic
- [ ] Valider les données côté application ET base
- [ ] Logger les erreurs sans exposer les détails
- [ ] Utiliser des variables d'environnement
- [ ] Fermer les connexions gracefully
- [ ] Monitorer les performances
- [ ] Tester avec des données réelles
- [ ] Documenter les schémas

---

**Pourquoi ce module est crucial :** Une intégration mal faite peut annuler tous les bénéfices d'une base bien configurée. Les patterns appris ici s'appliquent à tous vos projets.

---

### Module 16 : Fonctionnalités Avancées
**Durée estimée : 14-18 heures**

Features MongoDB pour des use cases spécifiques.

#### 16.1 Change Streams
**Durée : 4-5 heures**

Réactivité temps réel.

**Ce que vous maîtriserez :**
- Principes et architecture des change streams
- Filtrage des événements
- Resume tokens pour la résilience
- Cas d'usage (notifications, sync, cache invalidation)

**Exemple :**
```javascript
const changeStream = db.collection('orders').watch([
  { $match: { 'fullDocument.status': 'pending' } }
]);

changeStream.on('change', async (change) => {
  console.log('New pending order:', change.fullDocument);
  await sendNotification(change.fullDocument);
});

// Résilience : resume après crash
const resumeToken = await getLastResumeToken();
const changeStream = db.collection('orders').watch([], {
  resumeAfter: resumeToken
});
```

**Use cases :**
- Live dashboards
- Microservices event bus
- Cache invalidation automatique
- Real-time notifications
- Data synchronization

---

#### 16.2 GridFS
**Durée : 2-3 heures**

Stockage de fichiers volumineux.

**Quand utiliser GridFS :**
- ✅ Fichiers > 16 MB (limite document)
- ✅ Besoin de streaming
- ✅ Métadonnées associées aux fichiers
- ✅ Versioning de fichiers

**Quand ne pas utiliser GridFS :**
- ❌ Fichiers < 16 MB (utilisez documents normaux avec base64)
- ❌ Besoin de CDN (utilisez S3 + CloudFront)
- ❌ Très haute performance requise (S3 est plus rapide)

**Exemple :**
```javascript
const bucket = new GridFSBucket(db, { bucketName: 'uploads' });

// Upload
fs.createReadStream('./video.mp4')
  .pipe(bucket.openUploadStream('video.mp4', {
    metadata: { userId: 'user123', type: 'video' }
  }));

// Download
bucket.openDownloadStreamByName('video.mp4')
  .pipe(fs.createWriteStream('./downloaded.mp4'));
```

---

#### 16.3 Capped Collections
**Durée : 1-2 heures**

Collections à taille fixe.

**Caractéristiques :**
- Taille maximum fixe
- FIFO automatique (les plus vieux documents sont supprimés)
- Insert-only (pas de update/delete)
- Performance optimale pour logs

**Use cases :**
- Logs applicatifs
- Event streams temporaires
- Cache avec expiration automatique

```javascript
db.createCollection('logs', {
  capped: true,
  size: 100000,  // 100 KB
  max: 5000      // 5000 documents max
});
```

---

#### 16.4 Time Series Collections
**Durée : 3-4 heures**

Optimisation pour données temporelles.

**MongoDB 5.0+ feature** pour données IoT, metrics, logs.

**Avantages :**
- Compression automatique (90% de réduction de taille typique)
- Queries optimisées pour time-series
- Expiration automatique (TTL)

**Exemple :**
```javascript
db.createCollection('sensor_data', {
  timeseries: {
    timeField: 'timestamp',
    metaField: 'metadata',
    granularity: 'seconds'
  }
});

// Insert
db.sensor_data.insertOne({
  timestamp: new Date(),
  metadata: { sensorId: 'sensor_1', location: 'warehouse_A' },
  temperature: 23.5,
  humidity: 65.2
});

// Query optimisée
db.sensor_data.find({
  timestamp: {
    $gte: ISODate('2024-12-01T00:00:00Z'),
    $lt: ISODate('2024-12-02T00:00:00Z')
  },
  'metadata.sensorId': 'sensor_1'
});
```

---

#### 16.5 Clustered Collections
**Durée : 1-2 heures**

MongoDB 5.3+ : Stockage ordonné par _id.

**Avantages :**
- Meilleure performance pour requêtes par _id
- Réduction de taille (pas d'index secondaire pour _id)
- Idéal pour time series avec _id comme timestamp

---

#### 16.6 Requêtes géospatiales avancées
**Durée : 3-4 heures**

Localisation et géographie.

**Ce que vous maîtriserez :**
- Stockage de coordonnées (GeoJSON)
- Index 2dsphere
- Queries : $near, $geoWithin, $geoIntersects
- Calcul de distances

**Exemple :**
```javascript
// Schema
{
  name: 'Central Park',
  location: {
    type: 'Point',
    coordinates: [-73.968285, 40.785091]  // [longitude, latitude]
  }
}

// Index
db.places.createIndex({ location: '2dsphere' });

// Query : Trouver lieux dans un rayon de 5km
db.places.find({
  location: {
    $near: {
      $geometry: {
        type: 'Point',
        coordinates: [-73.9857, 40.7484]  // Times Square
      },
      $maxDistance: 5000  // 5000 mètres
    }
  }
});
```

---

#### 16.7-16.9 Search et AI/ML
**Durée : 4-5 heures**

Features modernes pour recherche et intelligence artificielle.

**Full-Text Search ($text) :**
- Index texte MongoDB natif
- Recherche basique
- Limité vs Atlas Search

**Atlas Search :**
- Lucene intégré
- Recherche avancée (autocomplete, fuzzy, facets)
- Déjà couvert en Partie 6

**Vector Search :**
- Recherche sémantique
- Intégration AI/ML (embeddings)
- RAG (Retrieval-Augmented Generation)
- Déjà couvert en Partie 6

---

#### 16.10 MongoDB et GraphQL
**Durée : 2-3 heures**

API moderne avec GraphQL.

**Approches :**
1. GraphQL server custom (Apollo, etc.) + MongoDB driver
2. Atlas GraphQL (auto-generated)

**Exemple Apollo Server :**
```javascript
const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
  }

  type Query {
    user(id: ID!): User
    posts: [Post!]!
  }
`;

const resolvers = {
  Query: {
    user: async (_, { id }, { db }) => {
      return await db.collection('users').findOne({ _id: new ObjectId(id) });
    },
    posts: async (_, __, { db }) => {
      return await db.collection('posts').find().toArray();
    }
  },
  User: {
    posts: async (user, _, { db }) => {
      return await db.collection('posts').find({ authorId: user._id }).toArray();
    }
  }
};
```

**Avantages GraphQL :**
- Requêtes flexibles (client demande exactement ce dont il a besoin)
- Pas de over-fetching ou under-fetching
- Typage fort

---

**Pourquoi ce module est important :** Ces fonctionnalités avancées résolvent des problèmes spécifiques. Savoir quand et comment les utiliser vous distingue des développeurs juniors.

## 🎯 Progression pédagogique

Cette partie suit une logique **intégration → patterns → features avancées** :

```
Drivers → ODM → Patterns → Features avancées → Intégration complète
```

### Semaines 1-2 : Drivers et intégration de base
**Focus : Connexion et CRUD solides**

**Semaine 1 : Driver principal**
- Jours 1-2 : Setup et première connexion
- Jours 3-4 : CRUD operations
- Jours 5-7 : Connection pooling, error handling, retry logic

**Semaine 2 : ODM/ORM**
- Jours 1-3 : Mongoose (ou équivalent pour votre langage)
- Jours 4-5 : Schemas, validation, middlewares
- Jours 6-7 : Population et queries avancées

**Livrables :**
- Application CRUD complète
- Repository pattern implémenté
- Tests unitaires et d'intégration
- Error handling robuste

---

### Semaine 3 : Patterns et architecture
**Focus : Code production-ready**

**Jours 1-3 : Patterns applicatifs**
- Repository Pattern
- Service Layer
- Unit of Work (transactions)

**Jours 4-5 : Performance**
- Caching strategy
- Éviter N+1 queries
- Bulk operations

**Jours 6-7 : Tests et qualité**
- MongoDB Memory Server
- Integration tests
- Mocking strategies

**Livrables :**
- Architecture en couches
- Tests complets (> 80% coverage)
- Performance benchmarks

---

### Semaine 4 : Fonctionnalités avancées
**Focus : Use cases spécialisés**

**Jours 1-2 : Change Streams**
- Event-driven architecture
- Real-time features

**Jours 3-4 : GridFS et Time Series**
- File storage
- IoT/metrics data

**Jours 5-7 : Géospatial et Search**
- Location-based features
- Full-text search

**Livrables :**
- Feature temps réel (notifications)
- Upload de fichiers avec GridFS
- Feature de recherche géospatiale

---

**Rythme recommandé :** 2-3 heures par jour avec beaucoup de pratique hands-on.

## 🧠 Principes de développement fondamentaux

### 1. Connection pooling is not optional

> Une nouvelle connexion par requête est l'anti-pattern #1. Always use a connection pool.

**Impact :** 100-500ms de latence évitée par requête.

### 2. Fail fast, recover gracefully

> Les erreurs vont arriver. Gérez-les dès qu'elles se produisent, avec retry logic et fallbacks.

**Application :**
- Retry pour erreurs transitoires
- Circuit breaker pour pannes prolongées
- Fallback vers cache si DB down

### 3. Validate everywhere

> Validation côté client ET application ET base. Defense in depth.

**Layers :**
- Frontend : UX
- API : Sécurité
- MongoDB : Intégrité

### 4. Don't trust the client, ever

> Tout input utilisateur est potentiellement malveillant.

**Application :**
- Sanitize toutes les entrées
- Parameterized queries (automatique avec drivers)
- Rate limiting
- Input validation stricte

### 5. Test with real data

> Les bugs se cachent dans les edge cases réels, pas dans les données de test parfaites.

**Application :**
- Seed DB avec données réalistes
- Test avec datasets volumineux
- Simuler network issues, timeouts

### 6. Monitor in production

> Vous ne pouvez pas améliorer ce que vous ne mesurez pas.

**Métriques app :**
- Latence des requêtes MongoDB
- Nombre de queries par endpoint
- Taille des résultats
- Taux d'erreurs
- Pool connection usage

## 🚦 Validation des acquis

Avant de passer à la Partie 8, vous devez maîtriser :

### Checklist Intégration
- [ ] Je peux connecter mon application à MongoDB avec connection pooling
- [ ] Je gère correctement les erreurs et implémente retry logic
- [ ] Je comprends les options de connection string
- [ ] Je peux choisir entre driver natif et ODM selon le contexte
- [ ] J'ai implémenté le Repository Pattern
- [ ] Je sais éviter les anti-patterns (N+1, etc.)

### Checklist ODM (si utilisé)
- [ ] Je maîtrise Mongoose (ou équivalent pour mon langage)
- [ ] Je définis des schémas avec validation
- [ ] J'utilise les middlewares efficacement
- [ ] Je comprends la population et ses limites
- [ ] Je sais quand utiliser virtuals vs computed fields

### Checklist Patterns
- [ ] J'ai implémenté une architecture en couches
- [ ] J'utilise Dependency Injection
- [ ] Je gère les transactions avec Unit of Work
- [ ] J'ai implémenté du caching
- [ ] Je fais du bulk processing quand approprié

### Checklist Features avancées
- [ ] Je peux utiliser Change Streams pour du real-time
- [ ] Je comprends GridFS et ses use cases
- [ ] Je sais utiliser les Time Series Collections
- [ ] Je peux implémenter des features géospatiales
- [ ] Je maîtrise la recherche full-text

### Checklist Tests et qualité
- [ ] J'écris des tests unitaires avec mocking
- [ ] J'écris des tests d'intégration avec MongoDB
- [ ] Je teste les edge cases (erreurs, timeouts)
- [ ] Mon code coverage est > 70%
- [ ] Je documente mes APIs et schémas

**Objectif :** Cocher 85%+ de ces cases.

## 🎯 Projet pratique : Application full-stack

### Projet : Blog / CMS avec fonctionnalités avancées
**Durée : 30-35 heures**

**Objectif :** Construire une application complète démontrant toutes les compétences d'intégration.

**Stack suggérée :**
- Backend : Node.js + Express (ou votre langage préféré)
- Database : MongoDB (local ou Atlas)
- ODM : Mongoose
- Frontend : React (optionnel, focus sur le backend)
- Tests : Jest + MongoDB Memory Server

**Fonctionnalités :**

**Core (15h) :**
1. Authentication (JWT)
2. CRUD Posts (titre, contenu, auteur, tags)
3. Comments système
4. User profiles
5. Repository Pattern + Service Layer
6. Validation complète
7. Error handling robuste
8. Tests (unit + integration)

**Avancées (15h) :**
9. Change Streams pour notifications temps réel
10. GridFS pour upload d'images (> 16 MB)
11. Full-text search sur posts
12. Géospatial : "Posts près de moi"
13. Analytics avec Time Series (page views)
14. Caching avec Redis
15. Rate limiting
16. CI/CD pipeline

**Livrables :**
- Code source complet (GitHub)
- Documentation API (Swagger/OpenAPI)
- Tests avec > 80% coverage
- README avec setup instructions
- Architecture diagram
- Performance benchmarks

**Critères de validation :**
- ✅ Connection pooling correct
- ✅ Repository Pattern implémenté
- ✅ Gestion d'erreurs complète
- ✅ Change Streams fonctionnels
- ✅ GridFS pour images
- ✅ Search full-text performant
- ✅ Tests passants
- ✅ Documentation complète

**Compétences validées :**
- Intégration MongoDB complète
- Patterns applicatifs
- Features avancées
- Tests et qualité
- Documentation

Ce projet démontre une maîtrise production-ready de MongoDB dans le développement applicatif.

## 📊 Comparaison : Driver natif vs ODM

| Critère | Driver Natif | ODM (Mongoose, etc.) |
|---------|--------------|----------------------|
| **Courbe d'apprentissage** | Faible | Moyenne |
| **Verbosité** | Élevée | Faible |
| **Performance** | Optimale | Très bonne (léger overhead) |
| **Type safety** | Limitée | Excellente |
| **Validation** | Manuelle | Automatique |
| **Boilerplate** | Beaucoup | Peu |
| **Flexibilité** | Totale | Élevée mais contrainte |
| **Debugging** | Facile (transparent) | Parfois opaque |
| **Production-ready** | Oui | Oui |

**Recommandation :**
- **Startups/MVP** : ODM (productivité)
- **Microservices performants** : Driver natif
- **Applications traditionnelles** : ODM
- **Équipes TypeScript** : ODM avec types stricts

## 🌟 Conseils de développeur

### 1. KISS (Keep It Simple, Stupid)
Le code le plus simple est souvent le meilleur. N'over-engineer pas.

### 2. Write code for humans first
Le code est lu 10x plus qu'il n'est écrit. Clarté > cleverness.

### 3. Test early, test often
Les tests vous font gagner du temps, pas en perdre.

### 4. Document your decisions
Pourquoi avez-vous choisi cette approche ? Le vous du futur vous remerciera.

### 5. Refactor mercilessly
Le code qui ne s'améliore pas se dégrade. Refactoring continu.

### 6. Learn from production
Les meilleurs insights viennent de la prod. Monitoring + logs = or.

### 7. Security is not optional
Validez, sanitize, secure. Toujours.

### 8. Performance matters
Mais prématurément optimiser est le mal. Profile first, optimize second.

## 📚 Ressources complémentaires

### Documentation drivers
- [MongoDB Node.js Driver](https://www.mongodb.com/docs/drivers/node/)
- [PyMongo (Python)](https://pymongo.readthedocs.io/)
- [MongoDB Java Driver](https://www.mongodb.com/docs/drivers/java/)
- [MongoDB C# Driver](https://www.mongodb.com/docs/drivers/csharp/)

### ODM/ORM
- [Mongoose Documentation](https://mongoosejs.com/docs/)
- [Spring Data MongoDB](https://spring.io/projects/spring-data-mongodb)
- [MongoEngine (Python)](http://mongoengine.org/)

### Patterns et architecture
- *Clean Code* par Robert Martin
- *Refactoring* par Martin Fowler
- *Patterns of Enterprise Application Architecture* par Martin Fowler

### Testing
- [MongoDB Memory Server](https://github.com/nodkz/mongodb-memory-server)
- Jest, Mocha, PyTest selon votre langage

## 🚀 Et après ?

Une fois cette partie maîtrisée, vous serez un **développeur MongoDB complet**. Vous saurez :

- Intégrer MongoDB dans n'importe quelle application
- Utiliser les patterns et architectures appropriés
- Implémenter des features avancées (real-time, géospatial, etc.)
- Écrire du code production-ready avec tests
- Optimiser les performances applicatives

La **Partie 8** vous enseignera la performance et le tuning avancés, pour passer de "ça marche" à "ça marche à grande échelle".

La **Partie 9** couvrira les cas d'usage réels et les bonnes pratiques, consolidant toutes vos connaissances.

Mais d'abord, **maîtrisez cette Partie 7**. Une application mal intégrée est une application qui échouera en production, peu importe la qualité de votre infrastructure MongoDB.

**Votre code est le pont entre les utilisateurs et les données. Construisez-le solide.**

---

**Prêt à devenir un développeur MongoDB expert ? Allons-y ! 💻**

---

**Prochaine étape :** [Module 15 - Drivers et Intégration Applicative →](/15-drivers-integration-applicative/README.md)

---

*💡 Citation du jour : "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." - Martin Fowler*

⏭️ [Module 15 - Drivers et Intégration Applicative →](/15-drivers-integration-applicative/README.md)
