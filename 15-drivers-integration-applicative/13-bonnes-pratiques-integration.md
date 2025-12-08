🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.13 Bonnes pratiques d'intégration

## Introduction

L'intégration de MongoDB dans une application nécessite plus que la simple connexion à la base de données. Cette section présente les bonnes pratiques essentielles pour créer des applications robustes, performantes et maintenables avec MongoDB, quel que soit le langage utilisé.

### Principes fondamentaux

Les bonnes pratiques d'intégration reposent sur quatre piliers :

1. **Fiabilité** : Gestion des erreurs, reconnexion, transactions
2. **Performance** : Connection pooling, indexation, requêtes optimisées
3. **Sécurité** : Authentification, validation, sanitization
4. **Maintenabilité** : Architecture claire, logging, monitoring

---

## 1. Gestion des Connexions

### 1.1 Connection Pooling

Le connection pooling est **essentiel** pour les performances en production.

#### Node.js / Mongoose

```javascript
// ❌ MAUVAIS : Créer une nouvelle connexion à chaque requête
async function getUserBad(userId) {
    const mongoose = require('mongoose');
    await mongoose.connect(uri); // Nouvelle connexion !
    const user = await User.findById(userId);
    await mongoose.disconnect();
    return user;
}

// ✅ BON : Réutiliser la connexion avec pool
// config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
    await mongoose.connect(process.env.MONGODB_URI, {
        maxPoolSize: 10,      // Max 10 connexions
        minPoolSize: 5,       // Min 5 connexions maintenues
        socketTimeoutMS: 45000,
        serverSelectionTimeoutMS: 5000,
    });

    console.log('MongoDB Connected with pool');
};

// app.js
await connectDB(); // Une seule fois au démarrage

// Ensuite, toutes les requêtes utilisent le pool
async function getUser(userId) {
    return await User.findById(userId); // Utilise le pool
}
```

#### Python / Motor

```python
# ❌ MAUVAIS : Nouvelle connexion à chaque appel
async def get_user_bad(user_id):
    client = motor.motor_asyncio.AsyncIOMotorClient(uri)  # Nouvelle connexion !
    db = client.myapp
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    client.close()
    return user

# ✅ BON : Singleton avec pool
# config/database.py
import motor.motor_asyncio

class Database:
    client: motor.motor_asyncio.AsyncIOMotorClient = None

    @classmethod
    def get_client(cls):
        if cls.client is None:
            cls.client = motor.motor_asyncio.AsyncIOMotorClient(
                uri,
                maxPoolSize=10,
                minPoolSize=5,
                maxIdleTimeMS=30000
            )
        return cls.client

    @classmethod
    def get_database(cls):
        return cls.get_client().myapp

# Utilisation
async def get_user(user_id):
    db = Database.get_database()
    return await db.users.find_one({"_id": ObjectId(user_id)})
```

#### Java / Spring Data MongoDB

```java
// ✅ BON : Configuration du pool dans Spring
@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Override
    public MongoClient mongoClient() {
        ConnectionString connString = new ConnectionString(uri);

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connString)
            .applyToConnectionPoolSettings(builder ->
                builder
                    .maxSize(100)           // Pool maximum
                    .minSize(10)            // Pool minimum
                    .maxWaitTime(60, TimeUnit.SECONDS)
                    .maxConnectionIdleTime(30, TimeUnit.SECONDS)
            )
            .build();

        return MongoClients.create(settings);
    }
}

// Le pool est géré automatiquement par Spring
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public User getUser(String id) {
        return userRepository.findById(id).orElse(null);
    }
}
```

### 1.2 Gestion du cycle de vie

#### Démarrage et arrêt gracieux

```javascript
// Node.js - Arrêt gracieux
const mongoose = require('mongoose');

// Connexion au démarrage
async function startup() {
    try {
        await mongoose.connect(process.env.MONGODB_URI);
        console.log('MongoDB connected');
    } catch (error) {
        console.error('MongoDB connection error:', error);
        process.exit(1);
    }
}

// Arrêt propre
async function shutdown() {
    try {
        await mongoose.connection.close();
        console.log('MongoDB connection closed');
        process.exit(0);
    } catch (error) {
        console.error('Error closing MongoDB connection:', error);
        process.exit(1);
    }
}

// Écouter les signaux d'arrêt
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);

// Gérer les erreurs non catchées
process.on('unhandledRejection', (error) => {
    console.error('Unhandled rejection:', error);
    shutdown();
});

startup();
```

```python
# Python - Context manager
import motor.motor_asyncio
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    """Lifespan manager pour FastAPI"""
    # Startup
    client = motor.motor_asyncio.AsyncIOMotorClient(uri)
    app.state.mongodb_client = client
    app.state.mongodb = client.myapp

    try:
        await client.admin.command('ping')
        print("MongoDB connected")
    except Exception as e:
        print(f"MongoDB connection failed: {e}")
        raise

    yield  # L'application tourne ici

    # Shutdown
    client.close()
    print("MongoDB connection closed")

# FastAPI
from fastapi import FastAPI

app = FastAPI(lifespan=lifespan)
```

---

## 2. Gestion des Erreurs

### 2.1 Retry Logic

Les erreurs transitoires (réseau, élection du primary) nécessitent une logique de retry.

#### Pattern de retry universel

```javascript
// Node.js - Retry avec backoff exponentiel
async function withRetry(fn, maxRetries = 3, baseDelay = 100) {
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            return await fn();
        } catch (error) {
            // Vérifier si c'est une erreur transitoire
            const isTransient =
                error.name === 'MongoNetworkError' ||
                error.name === 'MongoTimeoutError' ||
                (error.code >= 11600 && error.code <= 11602);

            if (!isTransient || attempt === maxRetries) {
                throw error;
            }

            // Backoff exponentiel
            const delay = baseDelay * Math.pow(2, attempt);
            console.log(`Retry attempt ${attempt + 1}/${maxRetries} after ${delay}ms`);
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}

// Utilisation
async function createUser(userData) {
    return await withRetry(async () => {
        const user = new User(userData);
        return await user.save();
    });
}
```

```python
# Python - Décorateur de retry
import asyncio
from functools import wraps
from pymongo.errors import NetworkTimeout, AutoReconnect

def async_retry(max_retries=3, base_delay=0.1):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except (NetworkTimeout, AutoReconnect) as e:
                    if attempt == max_retries:
                        raise

                    delay = base_delay * (2 ** attempt)
                    print(f"Retry {attempt + 1}/{max_retries} after {delay}s")
                    await asyncio.sleep(delay)
        return wrapper
    return decorator

# Utilisation
@async_retry(max_retries=3)
async def create_user(user_data):
    result = await db.users.insert_one(user_data)
    return result.inserted_id
```

```java
// Java - Retry avec Spring Retry
@Service
public class UserService {

    @Retryable(
        value = {MongoException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    public User createUser(User user) {
        return userRepository.save(user);
    }

    @Recover
    public User recover(MongoException e, User user) {
        log.error("Failed to create user after retries: {}", e.getMessage());
        throw new ServiceException("User creation failed", e);
    }
}
```

### 2.2 Gestion des erreurs spécifiques

```typescript
// TypeScript - Error handling complet
import { MongoError } from 'mongodb';

class DatabaseError extends Error {
    constructor(message: string, public originalError?: Error) {
        super(message);
        this.name = 'DatabaseError';
    }
}

class DuplicateKeyError extends DatabaseError {
    constructor(field: string, originalError?: Error) {
        super(`Duplicate value for field: ${field}`, originalError);
        this.name = 'DuplicateKeyError';
    }
}

class ValidationError extends DatabaseError {
    constructor(errors: Record<string, string>, originalError?: Error) {
        super('Validation failed', originalError);
        this.name = 'ValidationError';
    }
}

async function handleMongoError(error: any): Promise<never> {
    // Duplicate key error (code 11000)
    if (error.code === 11000) {
        const field = Object.keys(error.keyPattern || {})[0] || 'unknown';
        throw new DuplicateKeyError(field, error);
    }

    // Validation error
    if (error.name === 'ValidationError') {
        const errors: Record<string, string> = {};
        Object.keys(error.errors).forEach(key => {
            errors[key] = error.errors[key].message;
        });
        throw new ValidationError(errors, error);
    }

    // Network errors
    if (error.name === 'MongoNetworkError') {
        throw new DatabaseError('Database connection failed', error);
    }

    // Generic error
    throw new DatabaseError('Database operation failed', error);
}

// Utilisation dans un controller Express
app.post('/api/users', async (req, res) => {
    try {
        const user = await createUser(req.body);
        res.status(201).json(user);
    } catch (error) {
        try {
            await handleMongoError(error);
        } catch (handledError) {
            if (handledError instanceof DuplicateKeyError) {
                res.status(409).json({ error: handledError.message });
            } else if (handledError instanceof ValidationError) {
                res.status(400).json({ error: handledError.message });
            } else {
                res.status(500).json({ error: 'Internal server error' });
            }
        }
    }
});
```

---

## 3. Sécurité

### 3.1 Validation et Sanitization

#### Ne jamais faire confiance aux données utilisateur

```javascript
// ❌ MAUVAIS : Injection NoSQL possible
app.get('/api/users', async (req, res) => {
    const username = req.query.username;
    // Si username = { $ne: null }, retourne tous les utilisateurs !
    const user = await User.findOne({ username });
    res.json(user);
});

// ✅ BON : Validation stricte
const Joi = require('joi');

const usernameSchema = Joi.string().alphanum().min(3).max(30).required();

app.get('/api/users', async (req, res) => {
    try {
        // Valider que username est une string
        const { error, value } = usernameSchema.validate(req.query.username);

        if (error) {
            return res.status(400).json({ error: 'Invalid username' });
        }

        const user = await User.findOne({ username: value });
        res.json(user);
    } catch (error) {
        res.status(500).json({ error: 'Server error' });
    }
});
```

```python
# Python - Validation avec Pydantic
from pydantic import BaseModel, Field, validator
from typing import Optional

class UserQuery(BaseModel):
    username: str = Field(..., min_length=3, max_length=30, regex=r'^[a-zA-Z0-9_]+$')
    email: Optional[str] = None

    @validator('email')
    def validate_email(cls, v):
        if v and '@' not in v:
            raise ValueError('Invalid email format')
        return v

# FastAPI endpoint
@app.get("/api/users")
async def get_users(query: UserQuery = Depends()):
    # query est déjà validé
    user = await db.users.find_one({"username": query.username})
    return user
```

```java
// Java - Bean Validation
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    public ResponseEntity<User> getUser(
        @Valid @RequestParam
        @Pattern(regexp = "^[a-zA-Z0-9_]{3,30}$")
        String username
    ) {
        User user = userService.findByUsername(username);
        return ResponseEntity.ok(user);
    }
}
```

### 3.2 Prévention des injections NoSQL

```javascript
// ❌ DANGER : Construction de requête dynamique
function buildQuery(filters) {
    const query = {};
    for (const [key, value] of Object.entries(filters)) {
        query[key] = value; // Injection possible !
    }
    return query;
}

// ✅ BON : Whitelist des champs autorisés
function buildQuerySafe(filters) {
    const ALLOWED_FIELDS = ['username', 'email', 'status'];
    const query = {};

    for (const field of ALLOWED_FIELDS) {
        if (field in filters && typeof filters[field] === 'string') {
            query[field] = filters[field];
        }
    }

    return query;
}

// ✅ MEILLEUR : Utiliser un schéma de validation
const querySchema = Joi.object({
    username: Joi.string().alphanum().max(30),
    email: Joi.string().email(),
    status: Joi.string().valid('active', 'inactive')
});

async function searchUsers(filters) {
    const { error, value } = querySchema.validate(filters, {
        stripUnknown: true // Supprimer les champs non définis
    });

    if (error) {
        throw new Error('Invalid query parameters');
    }

    return await User.find(value);
}
```

### 3.3 Secrets et credentials

```javascript
// ❌ MAUVAIS : Credentials en dur
const mongoose = require('mongoose');
mongoose.connect('mongodb://admin:password123@localhost:27017/myapp');

// ✅ BON : Variables d'environnement
require('dotenv').config();

const uri = process.env.MONGODB_URI;
if (!uri) {
    throw new Error('MONGODB_URI environment variable is not set');
}

mongoose.connect(uri);

// .env (ne jamais commiter ce fichier)
// MONGODB_URI=mongodb://admin:password123@localhost:27017/myapp

// .env.example (commiter celui-ci)
// MONGODB_URI=mongodb://username:password@host:port/database
```

```python
# Python - Configuration sécurisée
import os
from pydantic import BaseSettings, Field

class Settings(BaseSettings):
    mongodb_uri: str = Field(..., env='MONGODB_URI')
    mongodb_database: str = Field(..., env='MONGODB_DATABASE')

    class Config:
        env_file = '.env'
        env_file_encoding = 'utf-8'

settings = Settings()

# Utilisation
client = motor.motor_asyncio.AsyncIOMotorClient(settings.mongodb_uri)
db = client[settings.mongodb_database]
```

---

## 4. Performance

### 4.1 Indexation

```javascript
// ✅ Définir les index dans le schéma
const userSchema = new mongoose.Schema({
    email: {
        type: String,
        required: true,
        unique: true,
        index: true  // Index simple
    },
    username: {
        type: String,
        required: true,
        unique: true
    },
    status: String,
    createdAt: { type: Date, default: Date.now }
});

// Index composé
userSchema.index({ status: 1, createdAt: -1 });

// Index text pour recherche
userSchema.index({ username: 'text', bio: 'text' });

// ⚠️ IMPORTANT : Désactiver auto-index en production
if (process.env.NODE_ENV === 'production') {
    mongoose.set('autoIndex', false);
}

// Créer les index manuellement
async function createIndexes() {
    await User.createIndexes();
    console.log('Indexes created');
}
```

### 4.2 Projection - Sélectionner uniquement les champs nécessaires

```javascript
// ❌ MAUVAIS : Récupérer tous les champs
async function getUsers() {
    return await User.find();  // Password inclus !
}

// ✅ BON : Projection explicite
async function getUsersMinimal() {
    return await User
        .find()
        .select('username email createdAt')  // Seulement ces champs
        .lean();  // Convertir en POJO (plus rapide)
}

// ✅ Exclure des champs sensibles
async function getUsersPublic() {
    return await User
        .find()
        .select('-password -passwordResetToken')  // Exclure
        .lean();
}
```

```python
# Python - Projection
async def get_users_minimal():
    cursor = db.users.find(
        {},
        {
            "_id": 1,
            "username": 1,
            "email": 1,
            "password": 0  # Exclure explicitement
        }
    )
    return await cursor.to_list(length=100)
```

### 4.3 Pagination efficace

```javascript
// ❌ MAUVAIS : Pagination avec skip (lent pour grandes collections)
async function getUsersPage(page, limit) {
    const skip = (page - 1) * limit;
    return await User.find()
        .skip(skip)  // Inefficace pour grandes valeurs
        .limit(limit);
}

// ✅ BON : Cursor-based pagination
async function getUsersCursor(lastId = null, limit = 20) {
    const query = lastId ? { _id: { $gt: lastId } } : {};

    const users = await User.find(query)
        .sort({ _id: 1 })
        .limit(limit)
        .lean();

    const hasMore = users.length === limit;
    const nextCursor = hasMore ? users[users.length - 1]._id : null;

    return {
        data: users,
        nextCursor,
        hasMore
    };
}

// API endpoint
app.get('/api/users', async (req, res) => {
    const { cursor, limit = 20 } = req.query;
    const result = await getUsersCursor(cursor, parseInt(limit));
    res.json(result);
});
```

### 4.4 Batch operations

```javascript
// ❌ MAUVAIS : Boucle d'insertions
async function createUsersBad(usersData) {
    for (const userData of usersData) {
        await User.create(userData);  // N requêtes !
    }
}

// ✅ BON : Insertion en masse
async function createUsers(usersData) {
    return await User.insertMany(usersData, {
        ordered: false  // Continue même si erreur
    });
}

// ✅ BON : Mise à jour en masse
async function deactivateUsers(userIds) {
    return await User.updateMany(
        { _id: { $in: userIds } },
        { $set: { isActive: false } }
    );
}

// ✅ Bulk operations pour opérations mixtes
async function bulkUpdate(operations) {
    const bulk = User.collection.initializeUnorderedBulkOp();

    operations.forEach(op => {
        if (op.type === 'update') {
            bulk.find({ _id: op.id }).update({ $set: op.data });
        } else if (op.type === 'delete') {
            bulk.find({ _id: op.id }).remove();
        }
    });

    return await bulk.execute();
}
```

### 4.5 Caching

```javascript
// Redis pour cache
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

async function getUserCached(userId) {
    // Vérifier le cache
    const cached = await redis.get(`user:${userId}`);
    if (cached) {
        return JSON.parse(cached);
    }

    // Charger depuis MongoDB
    const user = await User.findById(userId).lean();

    if (user) {
        // Mettre en cache (5 minutes)
        await redis.setex(
            `user:${userId}`,
            300,
            JSON.stringify(user)
        );
    }

    return user;
}

// Invalider le cache lors de mise à jour
async function updateUser(userId, updates) {
    const user = await User.findByIdAndUpdate(
        userId,
        updates,
        { new: true }
    );

    // Invalider le cache
    await redis.del(`user:${userId}`);

    return user;
}
```

---

## 5. Architecture et Patterns

### 5.1 Repository Pattern

```typescript
// TypeScript - Repository Pattern
// repository/BaseRepository.ts
import { Model, Document, FilterQuery } from 'mongoose';

export abstract class BaseRepository<T extends Document> {
    constructor(protected model: Model<T>) {}

    async findById(id: string): Promise<T | null> {
        return this.model.findById(id).exec();
    }

    async findOne(filter: FilterQuery<T>): Promise<T | null> {
        return this.model.findOne(filter).exec();
    }

    async find(
        filter: FilterQuery<T> = {},
        options: { limit?: number; skip?: number } = {}
    ): Promise<T[]> {
        return this.model
            .find(filter)
            .limit(options.limit || 100)
            .skip(options.skip || 0)
            .exec();
    }

    async create(data: Partial<T>): Promise<T> {
        return this.model.create(data);
    }

    async update(id: string, data: Partial<T>): Promise<T | null> {
        return this.model
            .findByIdAndUpdate(id, data, { new: true })
            .exec();
    }

    async delete(id: string): Promise<boolean> {
        const result = await this.model.deleteOne({ _id: id }).exec();
        return result.deletedCount > 0;
    }

    async count(filter: FilterQuery<T> = {}): Promise<number> {
        return this.model.countDocuments(filter).exec();
    }
}

// repository/UserRepository.ts
import { User, IUser } from '../models/User';
import { BaseRepository } from './BaseRepository';

export class UserRepository extends BaseRepository<IUser> {
    constructor() {
        super(User);
    }

    async findByEmail(email: string): Promise<IUser | null> {
        return this.model.findOne({ email }).exec();
    }

    async findActive(): Promise<IUser[]> {
        return this.model.find({ isActive: true }).exec();
    }

    async existsByEmail(email: string): Promise<boolean> {
        return this.model.exists({ email }) !== null;
    }
}

// service/UserService.ts
import { UserRepository } from '../repository/UserRepository';
import { IUser } from '../models/User';

export class UserService {
    constructor(private userRepository: UserRepository) {}

    async registerUser(userData: Partial<IUser>): Promise<IUser> {
        // Vérifier si l'email existe
        const exists = await this.userRepository.existsByEmail(userData.email!);
        if (exists) {
            throw new Error('Email already registered');
        }

        // Créer l'utilisateur
        return this.userRepository.create(userData);
    }

    async getUser(userId: string): Promise<IUser> {
        const user = await this.userRepository.findById(userId);
        if (!user) {
            throw new Error('User not found');
        }
        return user;
    }
}
```

### 5.2 Unit of Work Pattern (Transactions)

```javascript
// UnitOfWork pour gérer les transactions
class UnitOfWork {
    constructor(mongooseConnection) {
        this.connection = mongooseConnection;
    }

    async withTransaction(callback) {
        const session = await this.connection.startSession();

        try {
            await session.startTransaction();

            const result = await callback(session);

            await session.commitTransaction();
            return result;

        } catch (error) {
            await session.abortTransaction();
            throw error;
        } finally {
            session.endSession();
        }
    }
}

// Utilisation
const uow = new UnitOfWork(mongoose.connection);

async function createOrderWithInventory(orderData) {
    return await uow.withTransaction(async (session) => {
        // Créer la commande
        const order = await Order.create([orderData], { session });

        // Décrémenter le stock
        for (const item of orderData.items) {
            await Product.findByIdAndUpdate(
                item.productId,
                { $inc: { stock: -item.quantity } },
                { session }
            );
        }

        // Débiter le compte
        await Account.findByIdAndUpdate(
            orderData.accountId,
            { $inc: { balance: -orderData.total } },
            { session }
        );

        return order[0];
    });
}
```

---

## 6. Logging et Monitoring

### 6.1 Logging structuré

```javascript
// Winston pour logging structuré
const winston = require('winston');

const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
    ),
    defaultMeta: { service: 'user-service' },
    transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' })
    ]
});

if (process.env.NODE_ENV !== 'production') {
    logger.add(new winston.transports.Console({
        format: winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
        )
    }));
}

// Logger les opérations MongoDB
async function createUser(userData) {
    const startTime = Date.now();

    try {
        logger.info('Creating user', { email: userData.email });

        const user = await User.create(userData);

        const duration = Date.now() - startTime;
        logger.info('User created successfully', {
            userId: user._id,
            duration,
            email: userData.email
        });

        return user;

    } catch (error) {
        const duration = Date.now() - startTime;
        logger.error('Failed to create user', {
            error: error.message,
            stack: error.stack,
            duration,
            email: userData.email
        });
        throw error;
    }
}
```

### 6.2 Monitoring des requêtes

```javascript
// Middleware pour logger les requêtes MongoDB
mongoose.set('debug', (collectionName, method, query, doc) => {
    logger.debug('MongoDB Query', {
        collection: collectionName,
        method,
        query: JSON.stringify(query),
        doc: doc ? JSON.stringify(doc) : undefined
    });
});

// Métriques avec Prometheus
const promClient = require('prom-client');

const mongoQueryDuration = new promClient.Histogram({
    name: 'mongodb_query_duration_seconds',
    help: 'Duration of MongoDB queries',
    labelNames: ['operation', 'collection']
});

const mongoErrors = new promClient.Counter({
    name: 'mongodb_errors_total',
    help: 'Total number of MongoDB errors',
    labelNames: ['operation', 'error_type']
});

// Wrapper pour tracker les métriques
async function withMetrics(operation, collection, fn) {
    const end = mongoQueryDuration.startTimer({ operation, collection });

    try {
        return await fn();
    } catch (error) {
        mongoErrors.inc({ operation, error_type: error.name });
        throw error;
    } finally {
        end();
    }
}

// Utilisation
async function findUser(userId) {
    return await withMetrics('findById', 'users', async () => {
        return await User.findById(userId);
    });
}
```

### 6.3 Health checks

```javascript
// Express health check endpoint
app.get('/health', async (req, res) => {
    const health = {
        status: 'ok',
        timestamp: new Date().toISOString(),
        checks: {}
    };

    try {
        // Vérifier la connexion MongoDB
        await mongoose.connection.db.admin().ping();
        health.checks.mongodb = {
            status: 'up',
            responseTime: '< 100ms'
        };
    } catch (error) {
        health.status = 'error';
        health.checks.mongodb = {
            status: 'down',
            error: error.message
        };
        return res.status(503).json(health);
    }

    // Vérifier le pool de connexions
    const poolStats = mongoose.connection.db.serverConfig.s.pool;
    health.checks.connectionPool = {
        available: poolStats.availableConnections,
        total: poolStats.totalConnections,
        inUse: poolStats.inUseConnections
    };

    res.json(health);
});
```

---

## 7. Testing

### 7.1 Tests avec base de données de test

```javascript
// Jest - Setup et teardown
// setup.js
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongoServer;

beforeAll(async () => {
    mongoServer = await MongoMemoryServer.create();
    const mongoUri = mongoServer.getUri();
    await mongoose.connect(mongoUri);
});

afterAll(async () => {
    await mongoose.disconnect();
    await mongoServer.stop();
});

afterEach(async () => {
    const collections = mongoose.connection.collections;
    for (const key in collections) {
        await collections[key].deleteMany({});
    }
});

// user.test.js
describe('User Service', () => {
    it('should create a user', async () => {
        const userData = {
            username: 'testuser',
            email: 'test@example.com',
            password: 'password123'
        };

        const user = await userService.createUser(userData);

        expect(user.username).toBe('testuser');
        expect(user.email).toBe('test@example.com');
        expect(user._id).toBeDefined();
    });

    it('should not create duplicate email', async () => {
        const userData = {
            username: 'testuser',
            email: 'test@example.com',
            password: 'password123'
        };

        await userService.createUser(userData);

        await expect(
            userService.createUser(userData)
        ).rejects.toThrow('Email already exists');
    });
});
```

```python
# Python - Tests avec pytest et mongomock
import pytest
from mongomock_motor import AsyncMongoMockClient

@pytest.fixture
async def mock_db():
    """Fixture pour base de données mock"""
    client = AsyncMongoMockClient()
    db = client.test_db
    yield db
    client.close()

@pytest.fixture
def user_repository(mock_db):
    """Fixture pour repository"""
    return UserRepository(mock_db)

@pytest.mark.asyncio
async def test_create_user(user_repository):
    """Test création utilisateur"""
    user_data = {
        "username": "testuser",
        "email": "test@example.com",
        "password": "password123"
    }

    user_id = await user_repository.create(user_data)
    assert user_id is not None

    user = await user_repository.find_by_id(user_id)
    assert user["username"] == "testuser"

@pytest.mark.asyncio
async def test_duplicate_email(user_repository):
    """Test email dupliqué"""
    user_data = {
        "username": "testuser",
        "email": "test@example.com",
        "password": "password123"
    }

    await user_repository.create(user_data)

    with pytest.raises(Exception):
        await user_repository.create(user_data)
```

### 7.2 Mocking pour tests unitaires

```javascript
// Jest - Mock Mongoose
jest.mock('../models/User');
const User = require('../models/User');

describe('User Service - Unit Tests', () => {
    beforeEach(() => {
        jest.clearAllMocks();
    });

    it('should get user by email', async () => {
        const mockUser = {
            _id: '123',
            username: 'testuser',
            email: 'test@example.com'
        };

        User.findOne.mockResolvedValue(mockUser);

        const user = await userService.getUserByEmail('test@example.com');

        expect(User.findOne).toHaveBeenCalledWith({
            email: 'test@example.com'
        });
        expect(user).toEqual(mockUser);
    });
});
```

---

## 8. Déploiement et Production

### 8.1 Configuration par environnement

```javascript
// config/database.js
const environments = {
    development: {
        uri: process.env.MONGODB_URI_DEV || 'mongodb://localhost:27017/myapp_dev',
        options: {
            maxPoolSize: 10,
            serverSelectionTimeoutMS: 5000,
            autoIndex: true  // OK en dev
        }
    },

    staging: {
        uri: process.env.MONGODB_URI_STAGING,
        options: {
            maxPoolSize: 50,
            serverSelectionTimeoutMS: 5000,
            autoIndex: false,
            retryWrites: true,
            w: 'majority'
        }
    },

    production: {
        uri: process.env.MONGODB_URI_PROD,
        options: {
            maxPoolSize: 100,
            minPoolSize: 20,
            serverSelectionTimeoutMS: 5000,
            socketTimeoutMS: 45000,
            autoIndex: false,  // IMPORTANT en production
            retryWrites: true,
            retryReads: true,
            w: 'majority',
            readPreference: 'primaryPreferred',
            ssl: true,
            compressors: ['snappy', 'zlib']
        }
    }
};

const env = process.env.NODE_ENV || 'development';
const config = environments[env];

if (!config.uri) {
    throw new Error(`MongoDB URI not configured for environment: ${env}`);
}

module.exports = config;
```

### 8.2 Gestion des migrations

```javascript
// migrations/001-add-email-index.js
module.exports = {
    async up(db) {
        await db.collection('users').createIndex(
            { email: 1 },
            { unique: true, name: 'email_unique_idx' }
        );
        console.log('Created email index');
    },

    async down(db) {
        await db.collection('users').dropIndex('email_unique_idx');
        console.log('Dropped email index');
    }
};

// migrate.js
const mongoose = require('mongoose');
const fs = require('fs').promises;
const path = require('path');

async function runMigrations() {
    await mongoose.connect(process.env.MONGODB_URI);

    const migrationsPath = path.join(__dirname, 'migrations');
    const files = await fs.readdir(migrationsPath);

    for (const file of files.sort()) {
        if (!file.endsWith('.js')) continue;

        const migration = require(path.join(migrationsPath, file));

        console.log(`Running migration: ${file}`);
        await migration.up(mongoose.connection.db);
    }

    console.log('All migrations completed');
    await mongoose.disconnect();
}

runMigrations().catch(console.error);
```

### 8.3 Checklist de production

```markdown
## Checklist MongoDB Production

### Configuration
- [ ] Connection string sécurisée (variables d'environnement)
- [ ] SSL/TLS activé
- [ ] Authentification configurée
- [ ] Connection pool dimensionné (max/min)
- [ ] Timeouts configurés
- [ ] Retry logic implémentée
- [ ] Auto-index désactivé

### Performance
- [ ] Index créés et testés
- [ ] Requêtes optimisées (explain)
- [ ] Projections utilisées
- [ ] Pagination cursor-based
- [ ] Cache implémenté (Redis)
- [ ] Batch operations pour masse

### Sécurité
- [ ] Validation des entrées
- [ ] Protection injection NoSQL
- [ ] Pas de credentials en dur
- [ ] Champs sensibles exclus (password)
- [ ] Rate limiting sur API
- [ ] IP whitelist configurée

### Monitoring
- [ ] Logging structuré
- [ ] Métriques Prometheus
- [ ] Alertes configurées
- [ ] Health check endpoint
- [ ] Connection pool monitoring
- [ ] Query performance tracking

### Haute Disponibilité
- [ ] Replica Set configuré
- [ ] Read preference définie
- [ ] Write concern approprié
- [ ] Backups automatisés
- [ ] Plan de disaster recovery
- [ ] Failover testé

### Testing
- [ ] Tests unitaires (>80% coverage)
- [ ] Tests d'intégration
- [ ] Tests de charge
- [ ] Tests de failover
```

---

## 9. Anti-patterns à éviter

### 9.1 Anti-pattern : Documents trop volumineux

```javascript
// ❌ MAUVAIS : Document de 20 MB
const postSchema = new mongoose.Schema({
    title: String,
    content: String,
    comments: [{  // Peut grandir indéfiniment !
        author: String,
        text: String,
        createdAt: Date
    }],
    likes: [String]  // Array sans limite !
});

// ✅ BON : Références ou collection séparée
const postSchema = new mongoose.Schema({
    title: String,
    content: String,
    commentsCount: { type: Number, default: 0 },
    likesCount: { type: Number, default: 0 }
});

const commentSchema = new mongoose.Schema({
    postId: { type: mongoose.Schema.Types.ObjectId, ref: 'Post' },
    author: String,
    text: String,
    createdAt: Date
});
```

### 9.2 Anti-pattern : Requêtes N+1

```javascript
// ❌ MAUVAIS : N+1 queries
async function getPostsWithAuthors() {
    const posts = await Post.find();

    for (const post of posts) {
        // N requêtes supplémentaires !
        post.author = await User.findById(post.authorId);
    }

    return posts;
}

// ✅ BON : Population ou lookup
async function getPostsWithAuthorsGood() {
    return await Post.find().populate('author', 'username email');
}

// ✅ MEILLEUR : Agrégation pour contrôle total
async function getPostsWithAuthorsAggregation() {
    return await Post.aggregate([
        {
            $lookup: {
                from: 'users',
                localField: 'authorId',
                foreignField: '_id',
                as: 'author'
            }
        },
        { $unwind: '$author' },
        {
            $project: {
                title: 1,
                content: 1,
                'author.username': 1,
                'author.email': 1
            }
        }
    ]);
}
```

### 9.3 Anti-pattern : Pas de gestion d'erreur

```javascript
// ❌ MAUVAIS : Pas de gestion d'erreur
app.post('/api/users', async (req, res) => {
    const user = await User.create(req.body);  // Peut crasher !
    res.json(user);
});

// ✅ BON : Try/catch et logging
app.post('/api/users', async (req, res) => {
    try {
        const user = await User.create(req.body);
        logger.info('User created', { userId: user._id });
        res.status(201).json(user);
    } catch (error) {
        logger.error('Failed to create user', {
            error: error.message,
            body: req.body
        });

        if (error.code === 11000) {
            res.status(409).json({ error: 'User already exists' });
        } else if (error.name === 'ValidationError') {
            res.status(400).json({ error: error.message });
        } else {
            res.status(500).json({ error: 'Internal server error' });
        }
    }
});
```

---

## 10. Checklist des Bonnes Pratiques

### Connexions
- ✅ Utiliser le connection pooling
- ✅ Singleton pour la connexion
- ✅ Fermeture gracieuse
- ✅ Health checks

### Sécurité
- ✅ Variables d'environnement pour credentials
- ✅ Validation stricte des entrées
- ✅ Protection injection NoSQL
- ✅ SSL/TLS en production

### Performance
- ✅ Index appropriés
- ✅ Projections pour limiter les données
- ✅ Pagination cursor-based
- ✅ Batch operations
- ✅ lean() pour lecture seule
- ✅ Cache pour données fréquentes

### Erreurs
- ✅ Retry logic pour erreurs transitoires
- ✅ Try/catch systématique
- ✅ Logging structuré
- ✅ Error handling global

### Architecture
- ✅ Repository pattern
- ✅ Séparation des préoccupations
- ✅ Validation au niveau métier
- ✅ Tests unitaires et d'intégration

### Production
- ✅ Configuration par environnement
- ✅ Monitoring et alerting
- ✅ Backups automatisés
- ✅ Documentation à jour

---

## Conclusion

Les bonnes pratiques d'intégration MongoDB sont essentielles pour créer des applications robustes et performantes. Les points clés à retenir :

1. **Connection pooling** : Toujours réutiliser les connexions
2. **Gestion d'erreurs** : Retry logic et error handling complet
3. **Sécurité** : Validation, sanitization, variables d'environnement
4. **Performance** : Indexation, projection, pagination efficace
5. **Architecture** : Repository pattern, séparation des responsabilités
6. **Monitoring** : Logging, métriques, health checks
7. **Tests** : Coverage élevé, tests d'intégration

### Ressources complémentaires

- [MongoDB Best Practices](https://www.mongodb.com/docs/manual/administration/production-notes/)
- [Node.js Driver Best Practices](https://www.mongodb.com/docs/drivers/node/current/fundamentals/connection/)
- [Spring Data MongoDB Best Practices](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/)
- [Twelve-Factor App](https://12factor.net/)

---

**Prochaines sections** :
- 16.1 Change Streams - Streaming en temps réel
- 17.1 Performance et Tuning - Optimisations avancées

⏭️ [Fonctionnalités Avancées](/16-fonctionnalites-avancees/README.md)
