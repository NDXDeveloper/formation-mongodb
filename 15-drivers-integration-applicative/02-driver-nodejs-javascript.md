🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.2 Driver Node.js / JavaScript

## Introduction

Le driver Node.js MongoDB est l'un des drivers les plus populaires et matures de l'écosystème MongoDB. Il offre une API moderne basée sur les Promises, un excellent support TypeScript, et s'intègre parfaitement avec l'écosystème JavaScript/Node.js. Cette section explore en profondeur son utilisation pour des applications de niveau production.

## Installation et configuration

### Installation de base

```bash
# NPM
npm install mongodb

# Yarn
yarn add mongodb

# PNPM
pnpm add mongodb

# Version recommandée : 6.x (dernière stable)
# Node.js minimum : 16.20.1+
```

### Configuration TypeScript

```bash
# Le driver inclut les types TypeScript nativement
npm install --save-dev typescript @types/node

# tsconfig.json recommandé
```

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Imports

```javascript
// CommonJS (Node.js traditionnel)
const { MongoClient, ObjectId, ServerApiVersion } = require('mongodb');

// ES Modules (Node.js moderne)
import { MongoClient, ObjectId, ServerApiVersion } from 'mongodb';

// TypeScript
import {
    MongoClient,
    Db,
    Collection,
    ObjectId,
    Document,
    Filter,
    UpdateFilter,
    FindOptions,
    InsertOneResult,
    UpdateResult,
    DeleteResult
} from 'mongodb';
```

## Architecture de connexion

### Pattern Singleton (recommandé pour production)

```typescript
// src/database/mongodb.ts
import { MongoClient, Db, MongoClientOptions } from 'mongodb';

class MongoDBConnection {
    private static instance: MongoDBConnection;
    private client: MongoClient;
    private db!: Db;

    private constructor(uri: string, options?: MongoClientOptions) {
        this.client = new MongoClient(uri, {
            maxPoolSize: 50,
            minPoolSize: 10,
            maxIdleTimeMS: 30000,
            serverSelectionTimeoutMS: 5000,
            socketTimeoutMS: 45000,
            family: 4, // Force IPv4
            retryWrites: true,
            retryReads: true,
            ...options
        });
    }

    static getInstance(uri: string, options?: MongoClientOptions): MongoDBConnection {
        if (!MongoDBConnection.instance) {
            MongoDBConnection.instance = new MongoDBConnection(uri, options);
        }
        return MongoDBConnection.instance;
    }

    async connect(databaseName: string): Promise<void> {
        try {
            await this.client.connect();
            this.db = this.client.db(databaseName);

            // Vérifier la connexion
            await this.db.command({ ping: 1 });
            console.log('✅ Connecté à MongoDB');
        } catch (error) {
            console.error('❌ Erreur de connexion MongoDB:', error);
            throw error;
        }
    }

    getDatabase(): Db {
        if (!this.db) {
            throw new Error('Database not initialized. Call connect() first.');
        }
        return this.db;
    }

    getClient(): MongoClient {
        return this.client;
    }

    async close(): Promise<void> {
        await this.client.close();
        console.log('🔌 Connexion MongoDB fermée');
    }
}

// Export singleton
export const mongodb = MongoDBConnection.getInstance(
    process.env.MONGODB_URI || 'mongodb://localhost:27017'
);

export default mongodb;
```

```typescript
// src/index.ts - Utilisation
import mongodb from './database/mongodb';

async function main() {
    try {
        // Connexion au démarrage
        await mongodb.connect('myapp');

        const db = mongodb.getDatabase();
        const users = db.collection('users');

        // Votre application...

    } catch (error) {
        console.error('Erreur:', error);
        process.exit(1);
    }
}

// Fermeture propre
process.on('SIGINT', async () => {
    await mongodb.close();
    process.exit(0);
});

main();
```

### Configuration avancée avec environnements

```typescript
// src/config/database.config.ts
import { MongoClientOptions } from 'mongodb';

interface DatabaseConfig {
    uri: string;
    database: string;
    options: MongoClientOptions;
}

const configs: Record<string, DatabaseConfig> = {
    development: {
        uri: process.env.MONGODB_URI || 'mongodb://localhost:27017',
        database: 'myapp_dev',
        options: {
            maxPoolSize: 10,
            minPoolSize: 2,
            serverSelectionTimeoutMS: 5000,
            socketTimeoutMS: 45000,
            retryWrites: true,
            retryReads: true
        }
    },

    test: {
        uri: process.env.MONGODB_TEST_URI || 'mongodb://localhost:27017',
        database: 'myapp_test',
        options: {
            maxPoolSize: 5,
            minPoolSize: 1,
            serverSelectionTimeoutMS: 3000,
            socketTimeoutMS: 30000
        }
    },

    production: {
        uri: process.env.MONGODB_URI!,
        database: process.env.MONGODB_DATABASE || 'myapp',
        options: {
            maxPoolSize: 100,
            minPoolSize: 20,
            maxIdleTimeMS: 60000,
            serverSelectionTimeoutMS: 5000,
            socketTimeoutMS: 45000,
            compressors: ['snappy', 'zlib'],
            retryWrites: true,
            retryReads: true,
            readPreference: 'secondaryPreferred',
            w: 'majority',
            journal: true
        }
    }
};

const env = process.env.NODE_ENV || 'development';
export const dbConfig = configs[env];
```

## Types TypeScript avancés

### Définition des modèles

```typescript
// src/models/user.model.ts
import { ObjectId } from 'mongodb';

export interface UserBase {
    name: string;
    email: string;
    age?: number;
    status: 'active' | 'inactive' | 'suspended';
    roles: string[];
    preferences?: {
        language: string;
        timezone: string;
        notifications: boolean;
    };
}

export interface UserDocument extends UserBase {
    _id: ObjectId;
    createdAt: Date;
    updatedAt: Date;
    lastLogin?: Date;
    loginCount: number;
}

export type CreateUserDTO = Omit<UserDocument, '_id' | 'createdAt' | 'updatedAt' | 'loginCount'>;
export type UpdateUserDTO = Partial<Omit<UserDocument, '_id' | 'createdAt'>>;

// Pour les requêtes avec projection
export interface UserPublic {
    _id: ObjectId;
    name: string;
    email: string;
    status: string;
}
```

```typescript
// src/models/product.model.ts
import { ObjectId } from 'mongodb';

export interface Price {
    amount: number;
    currency: string;
    discount?: {
        percentage: number;
        validUntil: Date;
    };
}

export interface ProductDocument {
    _id: ObjectId;
    name: string;
    description: string;
    price: Price;
    category: string;
    tags: string[];
    inventory: {
        quantity: number;
        warehouse: string;
    };
    images: string[];
    reviews: {
        rating: number;
        count: number;
    };
    createdAt: Date;
    updatedAt: Date;
}
```

## Repository Pattern (recommandé)

### Repository de base

```typescript
// src/repositories/base.repository.ts
import {
    Collection,
    Document,
    Filter,
    OptionalUnlessRequiredId,
    UpdateFilter,
    FindOptions,
    InsertOneResult,
    UpdateResult,
    DeleteResult,
    ObjectId
} from 'mongodb';

export abstract class BaseRepository<T extends Document> {
    protected collection: Collection<T>;

    constructor(collection: Collection<T>) {
        this.collection = collection;
    }

    async findById(id: string | ObjectId): Promise<T | null> {
        const objectId = typeof id === 'string' ? new ObjectId(id) : id;
        return await this.collection.findOne({ _id: objectId } as Filter<T>);
    }

    async findOne(filter: Filter<T>): Promise<T | null> {
        return await this.collection.findOne(filter);
    }

    async findMany(
        filter: Filter<T> = {},
        options?: FindOptions<T>
    ): Promise<T[]> {
        return await this.collection.find(filter, options).toArray();
    }

    async create(doc: OptionalUnlessRequiredId<T>): Promise<InsertOneResult<T>> {
        return await this.collection.insertOne(doc);
    }

    async update(
        filter: Filter<T>,
        update: UpdateFilter<T>
    ): Promise<UpdateResult<T>> {
        return await this.collection.updateOne(filter, update);
    }

    async updateById(
        id: string | ObjectId,
        update: UpdateFilter<T>
    ): Promise<UpdateResult<T>> {
        const objectId = typeof id === 'string' ? new ObjectId(id) : id;
        return await this.collection.updateOne(
            { _id: objectId } as Filter<T>,
            update
        );
    }

    async delete(filter: Filter<T>): Promise<DeleteResult> {
        return await this.collection.deleteOne(filter);
    }

    async deleteById(id: string | ObjectId): Promise<DeleteResult> {
        const objectId = typeof id === 'string' ? new ObjectId(id) : id;
        return await this.collection.deleteOne({ _id: objectId } as Filter<T>);
    }

    async count(filter: Filter<T> = {}): Promise<number> {
        return await this.collection.countDocuments(filter);
    }

    async exists(filter: Filter<T>): Promise<boolean> {
        const count = await this.collection.countDocuments(filter, { limit: 1 });
        return count > 0;
    }
}
```

### Repository spécifique

```typescript
// src/repositories/user.repository.ts
import { Collection, ObjectId } from 'mongodb';
import { BaseRepository } from './base.repository';
import { UserDocument, CreateUserDTO, UpdateUserDTO } from '../models/user.model';

export class UserRepository extends BaseRepository<UserDocument> {
    constructor(collection: Collection<UserDocument>) {
        super(collection);
    }

    async createUser(userData: CreateUserDTO): Promise<UserDocument> {
        const now = new Date();
        const userDoc: OptionalUnlessRequiredId<UserDocument> = {
            ...userData,
            createdAt: now,
            updatedAt: now,
            loginCount: 0
        };

        const result = await this.create(userDoc);

        return {
            _id: result.insertedId,
            ...userDoc
        } as UserDocument;
    }

    async findByEmail(email: string): Promise<UserDocument | null> {
        return await this.findOne({ email });
    }

    async findActiveUsers(limit: number = 100): Promise<UserDocument[]> {
        return await this.findMany(
            { status: 'active' },
            {
                limit,
                sort: { createdAt: -1 }
            }
        );
    }

    async updateProfile(
        userId: string | ObjectId,
        updates: UpdateUserDTO
    ): Promise<boolean> {
        const result = await this.updateById(userId, {
            $set: {
                ...updates,
                updatedAt: new Date()
            }
        });

        return result.modifiedCount > 0;
    }

    async incrementLoginCount(userId: string | ObjectId): Promise<void> {
        await this.updateById(userId, {
            $inc: { loginCount: 1 },
            $set: { lastLogin: new Date(), updatedAt: new Date() }
        });
    }

    async searchUsers(searchTerm: string, limit: number = 20): Promise<UserDocument[]> {
        return await this.findMany(
            {
                $or: [
                    { name: { $regex: searchTerm, $options: 'i' } },
                    { email: { $regex: searchTerm, $options: 'i' } }
                ]
            },
            { limit }
        );
    }

    async getUsersByRole(role: string): Promise<UserDocument[]> {
        return await this.findMany({ roles: role });
    }

    async bulkUpdateStatus(
        userIds: ObjectId[],
        status: UserDocument['status']
    ): Promise<number> {
        const result = await this.collection.updateMany(
            { _id: { $in: userIds } },
            { $set: { status, updatedAt: new Date() } }
        );

        return result.modifiedCount;
    }

    async getStatistics(): Promise<{
        total: number;
        byStatus: Record<string, number>;
        recentSignups: number;
    }> {
        const [total, byStatus, recentSignups] = await Promise.all([
            this.count(),
            this.collection.aggregate([
                { $group: { _id: '$status', count: { $sum: 1 } } }
            ]).toArray(),
            this.count({
                createdAt: {
                    $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
                }
            })
        ]);

        const statusMap = byStatus.reduce((acc, { _id, count }) => {
            acc[_id] = count;
            return acc;
        }, {} as Record<string, number>);

        return {
            total,
            byStatus: statusMap,
            recentSignups
        };
    }
}
```

### Utilisation des repositories

```typescript
// src/services/user.service.ts
import { ObjectId } from 'mongodb';
import { UserRepository } from '../repositories/user.repository';
import { CreateUserDTO, UpdateUserDTO } from '../models/user.model';

export class UserService {
    constructor(private userRepository: UserRepository) {}

    async registerUser(userData: CreateUserDTO) {
        // Vérifier si l'email existe déjà
        const existingUser = await this.userRepository.findByEmail(userData.email);
        if (existingUser) {
            throw new Error('Email already registered');
        }

        // Créer l'utilisateur
        const user = await this.userRepository.createUser(userData);

        // Logique métier supplémentaire (envoi email, etc.)
        // ...

        return user;
    }

    async loginUser(email: string) {
        const user = await this.userRepository.findByEmail(email);
        if (!user) {
            throw new Error('User not found');
        }

        if (user.status !== 'active') {
            throw new Error('Account is not active');
        }

        // Incrémenter le compteur de connexions
        await this.userRepository.incrementLoginCount(user._id);

        return user;
    }

    async updateUserProfile(userId: string, updates: UpdateUserDTO) {
        const success = await this.userRepository.updateProfile(userId, updates);
        if (!success) {
            throw new Error('Failed to update user profile');
        }
    }

    async searchUsers(query: string) {
        return await this.userRepository.searchUsers(query);
    }

    async getDashboardStats() {
        return await this.userRepository.getStatistics();
    }
}
```

## Opérations CRUD avancées

### Insertions avec validation

```typescript
// Insertion avec validation et transformation
async function createProduct(productData: any): Promise<ProductDocument> {
    const collection = db.collection<ProductDocument>('products');

    // Validation
    if (!productData.name || productData.name.trim().length === 0) {
        throw new Error('Product name is required');
    }

    if (!productData.price || productData.price.amount <= 0) {
        throw new Error('Valid price is required');
    }

    // Transformation et enrichissement
    const product: OptionalUnlessRequiredId<ProductDocument> = {
        name: productData.name.trim(),
        description: productData.description || '',
        price: {
            amount: productData.price.amount,
            currency: productData.price.currency || 'EUR'
        },
        category: productData.category,
        tags: productData.tags || [],
        inventory: {
            quantity: productData.inventory?.quantity || 0,
            warehouse: productData.inventory?.warehouse || 'main'
        },
        images: productData.images || [],
        reviews: {
            rating: 0,
            count: 0
        },
        createdAt: new Date(),
        updatedAt: new Date()
    };

    const result = await collection.insertOne(product);

    return {
        _id: result.insertedId,
        ...product
    } as ProductDocument;
}
```

```typescript
// Insertion multiple avec gestion d'erreurs
async function bulkCreateProducts(
    productsData: any[]
): Promise<{
    success: ProductDocument[];
    errors: Array<{ index: number; error: string }>;
}> {
    const collection = db.collection<ProductDocument>('products');
    const success: ProductDocument[] = [];
    const errors: Array<{ index: number; error: string }> = [];

    // Option 1: Insertion ordered (s'arrête au premier échec)
    try {
        const docs = productsData.map(data => ({
            ...data,
            createdAt: new Date(),
            updatedAt: new Date()
        }));

        const result = await collection.insertMany(docs, { ordered: true });

        Object.values(result.insertedIds).forEach((id, index) => {
            success.push({ _id: id, ...docs[index] } as ProductDocument);
        });

    } catch (error: any) {
        if (error.writeErrors) {
            error.writeErrors.forEach((err: any) => {
                errors.push({ index: err.index, error: err.errmsg });
            });
        }
    }

    return { success, errors };
}
```

### Recherches complexes

```typescript
// Recherche avec filtres multiples et pagination
interface SearchOptions {
    page?: number;
    limit?: number;
    sortBy?: string;
    sortOrder?: 'asc' | 'desc';
}

async function searchProducts(
    filters: {
        category?: string;
        minPrice?: number;
        maxPrice?: number;
        tags?: string[];
        inStock?: boolean;
        searchTerm?: string;
    },
    options: SearchOptions = {}
): Promise<{
    data: ProductDocument[];
    total: number;
    page: number;
    totalPages: number;
}> {
    const collection = db.collection<ProductDocument>('products');

    // Construction du filtre
    const filter: Filter<ProductDocument> = {};

    if (filters.category) {
        filter.category = filters.category;
    }

    if (filters.minPrice !== undefined || filters.maxPrice !== undefined) {
        filter['price.amount'] = {};
        if (filters.minPrice !== undefined) {
            filter['price.amount'].$gte = filters.minPrice;
        }
        if (filters.maxPrice !== undefined) {
            filter['price.amount'].$lte = filters.maxPrice;
        }
    }

    if (filters.tags && filters.tags.length > 0) {
        filter.tags = { $all: filters.tags };
    }

    if (filters.inStock) {
        filter['inventory.quantity'] = { $gt: 0 };
    }

    if (filters.searchTerm) {
        filter.$or = [
            { name: { $regex: filters.searchTerm, $options: 'i' } },
            { description: { $regex: filters.searchTerm, $options: 'i' } }
        ];
    }

    // Pagination
    const page = options.page || 1;
    const limit = options.limit || 20;
    const skip = (page - 1) * limit;

    // Tri
    const sortField = options.sortBy || 'createdAt';
    const sortOrder = options.sortOrder === 'asc' ? 1 : -1;

    // Exécution parallèle du count et de la recherche
    const [data, total] = await Promise.all([
        collection
            .find(filter)
            .sort({ [sortField]: sortOrder })
            .skip(skip)
            .limit(limit)
            .toArray(),
        collection.countDocuments(filter)
    ]);

    return {
        data,
        total,
        page,
        totalPages: Math.ceil(total / limit)
    };
}
```

```typescript
// Recherche avec projection pour optimiser la performance
async function getProductSummaries(): Promise<Array<{
    _id: ObjectId;
    name: string;
    price: number;
    category: string;
}>> {
    const collection = db.collection<ProductDocument>('products');

    return await collection
        .find(
            { 'inventory.quantity': { $gt: 0 } },
            {
                projection: {
                    name: 1,
                    'price.amount': 1,
                    category: 1,
                    // Exclusion explicite de champs volumineux
                    description: 0,
                    images: 0,
                    reviews: 0
                }
            }
        )
        .toArray() as any;
}
```

### Mises à jour avancées

```typescript
// Mise à jour avec opérateurs multiples
async function updateProductInventory(
    productId: ObjectId,
    quantityChange: number,
    operation: 'add' | 'subtract'
): Promise<boolean> {
    const collection = db.collection<ProductDocument>('products');

    const update: UpdateFilter<ProductDocument> = {
        $inc: {
            'inventory.quantity': operation === 'add' ? quantityChange : -quantityChange
        },
        $set: {
            updatedAt: new Date()
        }
    };

    // Empêcher les quantités négatives
    const filter: Filter<ProductDocument> = {
        _id: productId
    };

    if (operation === 'subtract') {
        filter['inventory.quantity'] = { $gte: quantityChange };
    }

    const result = await collection.updateOne(filter, update);

    return result.modifiedCount > 0;
}
```

```typescript
// Mise à jour avec upsert (insert si n'existe pas)
async function updateOrCreateUserPreferences(
    userId: ObjectId,
    preferences: UserDocument['preferences']
): Promise<void> {
    const collection = db.collection<UserDocument>('users');

    await collection.updateOne(
        { _id: userId },
        {
            $set: {
                preferences,
                updatedAt: new Date()
            },
            $setOnInsert: {
                createdAt: new Date(),
                loginCount: 0,
                status: 'active',
                roles: ['user']
            }
        },
        { upsert: true }
    );
}
```

```typescript
// Mise à jour conditionnelle avec arrayFilters
async function updateSpecificReview(
    productId: ObjectId,
    reviewId: string,
    newRating: number
): Promise<boolean> {
    const collection = db.collection('products');

    const result = await collection.updateOne(
        { _id: productId },
        {
            $set: {
                'reviews.$[review].rating': newRating,
                'reviews.$[review].updatedAt': new Date()
            }
        },
        {
            arrayFilters: [{ 'review.id': reviewId }]
        }
    );

    return result.modifiedCount > 0;
}
```

### Suppressions conditionnelles

```typescript
// Suppression douce (soft delete)
async function softDeleteUser(userId: ObjectId): Promise<boolean> {
    const collection = db.collection<UserDocument>('users');

    const result = await collection.updateOne(
        { _id: userId, status: { $ne: 'deleted' } },
        {
            $set: {
                status: 'deleted',
                deletedAt: new Date(),
                updatedAt: new Date()
            }
        }
    );

    return result.modifiedCount > 0;
}
```

```typescript
// Suppression en masse avec limite
async function cleanupOldInactiveUsers(daysInactive: number): Promise<number> {
    const collection = db.collection<UserDocument>('users');

    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysInactive);

    const result = await collection.deleteMany({
        status: 'inactive',
        lastLogin: { $lt: cutoffDate }
    });

    console.log(`Supprimé ${result.deletedCount} utilisateurs inactifs`);
    return result.deletedCount;
}
```

## Agrégations

### Pipeline d'agrégation basique

```typescript
// Statistiques utilisateurs par statut
async function getUserStatsByStatus() {
    const collection = db.collection<UserDocument>('users');

    return await collection.aggregate([
        {
            $group: {
                _id: '$status',
                count: { $sum: 1 },
                avgAge: { $avg: '$age' },
                totalLogins: { $sum: '$loginCount' }
            }
        },
        {
            $sort: { count: -1 }
        }
    ]).toArray();
}
```

### Agrégation complexe avec jointures

```typescript
interface OrderWithUser {
    _id: ObjectId;
    orderNumber: string;
    amount: number;
    status: string;
    user: {
        name: string;
        email: string;
    };
    items: Array<{
        product: ProductDocument;
        quantity: number;
    }>;
}

async function getOrdersWithDetails(
    startDate: Date,
    endDate: Date
): Promise<OrderWithUser[]> {
    const collection = db.collection('orders');

    return await collection.aggregate<OrderWithUser>([
        // Filtrer par date
        {
            $match: {
                createdAt: { $gte: startDate, $lte: endDate }
            }
        },

        // Jointure avec users
        {
            $lookup: {
                from: 'users',
                localField: 'userId',
                foreignField: '_id',
                as: 'userDetails'
            }
        },

        // Dérouler le tableau userDetails
        {
            $unwind: '$userDetails'
        },

        // Dérouler les items pour la jointure
        {
            $unwind: '$items'
        },

        // Jointure avec products
        {
            $lookup: {
                from: 'products',
                localField: 'items.productId',
                foreignField: '_id',
                as: 'items.productDetails'
            }
        },

        // Regrouper les items
        {
            $group: {
                _id: '$_id',
                orderNumber: { $first: '$orderNumber' },
                amount: { $first: '$amount' },
                status: { $first: '$status' },
                user: {
                    $first: {
                        name: '$userDetails.name',
                        email: '$userDetails.email'
                    }
                },
                items: {
                    $push: {
                        product: { $arrayElemAt: ['$items.productDetails', 0] },
                        quantity: '$items.quantity'
                    }
                }
            }
        },

        // Tri
        {
            $sort: { orderNumber: -1 }
        }
    ]).toArray();
}
```

### Agrégation avec facettes

```typescript
// Statistiques multiples en une seule requête
async function getProductAnalytics() {
    const collection = db.collection<ProductDocument>('products');

    const result = await collection.aggregate([
        {
            $facet: {
                // Statistiques générales
                general: [
                    {
                        $group: {
                            _id: null,
                            totalProducts: { $sum: 1 },
                            avgPrice: { $avg: '$price.amount' },
                            totalInventory: { $sum: '$inventory.quantity' }
                        }
                    }
                ],

                // Par catégorie
                byCategory: [
                    {
                        $group: {
                            _id: '$category',
                            count: { $sum: 1 },
                            avgPrice: { $avg: '$price.amount' }
                        }
                    },
                    { $sort: { count: -1 } },
                    { $limit: 10 }
                ],

                // Distribution des prix
                priceDistribution: [
                    {
                        $bucket: {
                            groupBy: '$price.amount',
                            boundaries: [0, 10, 50, 100, 500, 1000, Infinity],
                            default: 'Other',
                            output: {
                                count: { $sum: 1 },
                                products: { $push: '$name' }
                            }
                        }
                    }
                ],

                // Top produits
                topRated: [
                    { $sort: { 'reviews.rating': -1 } },
                    { $limit: 10 },
                    {
                        $project: {
                            name: 1,
                            rating: '$reviews.rating',
                            reviewCount: '$reviews.count'
                        }
                    }
                ]
            }
        }
    ]).toArray();

    return result[0];
}
```

## Transactions

### Transaction simple

```typescript
async function transferFunds(
    fromUserId: ObjectId,
    toUserId: ObjectId,
    amount: number
): Promise<void> {
    const client = mongodb.getClient();
    const session = client.startSession();

    try {
        await session.withTransaction(async () => {
            const accounts = db.collection('accounts');

            // Débiter le compte source
            const debitResult = await accounts.updateOne(
                {
                    userId: fromUserId,
                    balance: { $gte: amount }
                },
                {
                    $inc: { balance: -amount },
                    $push: {
                        transactions: {
                            type: 'debit',
                            amount,
                            timestamp: new Date()
                        }
                    }
                },
                { session }
            );

            if (debitResult.modifiedCount === 0) {
                throw new Error('Insufficient funds or account not found');
            }

            // Créditer le compte destination
            await accounts.updateOne(
                { userId: toUserId },
                {
                    $inc: { balance: amount },
                    $push: {
                        transactions: {
                            type: 'credit',
                            amount,
                            timestamp: new Date()
                        }
                    }
                },
                { session }
            );
        });

        console.log('✅ Transaction completed successfully');

    } catch (error) {
        console.error('❌ Transaction failed:', error);
        throw error;
    } finally {
        await session.endSession();
    }
}
```

### Transaction complexe avec retry

```typescript
async function createOrderWithInventoryUpdate(
    orderData: any
): Promise<{ orderId: ObjectId; success: boolean }> {
    const client = mongodb.getClient();
    const maxRetries = 3;
    let lastError: any;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        const session = client.startSession();

        try {
            const result = await session.withTransaction(async () => {
                const orders = db.collection('orders');
                const products = db.collection<ProductDocument>('products');

                // 1. Vérifier et réserver l'inventaire
                for (const item of orderData.items) {
                    const updateResult = await products.updateOne(
                        {
                            _id: new ObjectId(item.productId),
                            'inventory.quantity': { $gte: item.quantity }
                        },
                        {
                            $inc: { 'inventory.quantity': -item.quantity },
                            $set: { updatedAt: new Date() }
                        },
                        { session }
                    );

                    if (updateResult.modifiedCount === 0) {
                        throw new Error(`Product ${item.productId} out of stock`);
                    }
                }

                // 2. Créer la commande
                const order = {
                    ...orderData,
                    status: 'pending',
                    createdAt: new Date(),
                    updatedAt: new Date()
                };

                const orderResult = await orders.insertOne(order, { session });

                // 3. Mettre à jour les statistiques utilisateur
                await db.collection('users').updateOne(
                    { _id: new ObjectId(orderData.userId) },
                    {
                        $inc: { 'stats.totalOrders': 1 },
                        $set: { 'stats.lastOrderDate': new Date() }
                    },
                    { session }
                );

                return orderResult.insertedId;
            }, {
                readConcern: { level: 'snapshot' },
                writeConcern: { w: 'majority' },
                readPreference: 'primary'
            });

            await session.endSession();

            return { orderId: result, success: true };

        } catch (error: any) {
            await session.endSession();
            lastError = error;

            // Retry uniquement sur les erreurs transitoires
            if (error.hasErrorLabel('TransientTransactionError') && attempt < maxRetries) {
                console.log(`Transaction failed (attempt ${attempt}), retrying...`);
                await new Promise(resolve => setTimeout(resolve, 100 * attempt));
                continue;
            }

            throw error;
        }
    }

    throw new Error(`Transaction failed after ${maxRetries} attempts: ${lastError.message}`);
}
```

## Change Streams

### Écoute des changements sur une collection

```typescript
async function watchUserChanges() {
    const collection = db.collection<UserDocument>('users');

    const changeStream = collection.watch(
        [
            // Filtrer uniquement les insertions et mises à jour
            { $match: { operationType: { $in: ['insert', 'update'] } } }
        ],
        {
            fullDocument: 'updateLookup' // Récupérer le document complet
        }
    );

    console.log('👀 Watching for changes...');

    changeStream.on('change', (change) => {
        console.log('📢 Change detected:', change.operationType);

        switch (change.operationType) {
            case 'insert':
                console.log('New user:', change.fullDocument);
                // Envoyer email de bienvenue, etc.
                break;

            case 'update':
                console.log('Updated user:', change.fullDocument);
                // Invalider cache, notifier, etc.
                break;

            case 'delete':
                console.log('Deleted user:', change.documentKey);
                break;
        }
    });

    changeStream.on('error', (error) => {
        console.error('❌ Change stream error:', error);
    });

    // Fermeture propre
    process.on('SIGINT', async () => {
        await changeStream.close();
        console.log('Change stream closed');
        process.exit(0);
    });
}
```

### Change Stream avec reprise après erreur

```typescript
class ChangeStreamManager {
    private changeStream?: ChangeStream;
    private resumeToken?: ResumeToken;

    async start() {
        const collection = db.collection('orders');

        const options: ChangeStreamOptions = {
            fullDocument: 'updateLookup',
            resumeAfter: this.resumeToken
        };

        this.changeStream = collection.watch([], options);

        this.changeStream.on('change', (change) => {
            // Sauvegarder le resume token
            this.resumeToken = change._id;

            this.handleChange(change);
        });

        this.changeStream.on('error', async (error) => {
            console.error('Change stream error:', error);

            // Fermer et redémarrer
            await this.changeStream?.close();

            // Attendre avant de redémarrer
            setTimeout(() => this.start(), 5000);
        });
    }

    private handleChange(change: ChangeStreamDocument) {
        console.log('Processing change:', change.operationType);

        // Traiter le changement
        // Par exemple: envoyer une notification, mettre à jour un cache, etc.
    }

    async stop() {
        await this.changeStream?.close();
    }
}
```

## Gestion d'erreurs avancée

### Wrapper avec retry automatique

```typescript
async function withRetry<T>(
    operation: () => Promise<T>,
    options: {
        maxRetries?: number;
        retryDelay?: number;
        retryableErrors?: string[];
    } = {}
): Promise<T> {
    const {
        maxRetries = 3,
        retryDelay = 1000,
        retryableErrors = ['MongoNetworkError', 'MongoServerError']
    } = options;

    let lastError: any;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            return await operation();
        } catch (error: any) {
            lastError = error;

            const isRetryable = retryableErrors.some(
                errorType => error.name === errorType
            );

            if (!isRetryable || attempt === maxRetries) {
                throw error;
            }

            console.log(`Attempt ${attempt} failed, retrying in ${retryDelay}ms...`);
            await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
        }
    }

    throw lastError;
}

// Utilisation
const user = await withRetry(
    () => userRepository.findById(userId),
    { maxRetries: 3, retryDelay: 500 }
);
```

### Gestionnaire d'erreurs centralisé

```typescript
class MongoDBErrorHandler {
    static handle(error: any): never {
        if (error.code === 11000) {
            // Duplicate key error
            const field = Object.keys(error.keyPattern)[0];
            throw new Error(`Duplicate value for field: ${field}`);
        }

        if (error.name === 'MongoNetworkError') {
            throw new Error('Database connection failed. Please try again.');
        }

        if (error.name === 'MongoServerError') {
            if (error.code === 50) {
                throw new Error('Operation exceeded time limit');
            }
        }

        if (error.name === 'BSONTypeError') {
            throw new Error('Invalid data format');
        }

        // Erreur non gérée
        console.error('Unhandled MongoDB error:', error);
        throw new Error('An unexpected database error occurred');
    }
}

// Utilisation
try {
    await collection.insertOne(document);
} catch (error) {
    MongoDBErrorHandler.handle(error);
}
```

## Performance et optimisation

### Connection pooling optimal

```typescript
// Configuration production recommandée
const client = new MongoClient(uri, {
    // Pool de connexions
    maxPoolSize: 100,
    minPoolSize: 20,
    maxConnecting: 10,
    maxIdleTimeMS: 60000,
    waitQueueTimeoutMS: 10000,

    // Timeouts
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    connectTimeoutMS: 10000,

    // Performance
    compressors: ['snappy', 'zlib'],
    zlibCompressionLevel: 6,

    // Retry
    retryWrites: true,
    retryReads: true,

    // Monitoring
    monitorCommands: process.env.NODE_ENV === 'development'
});
```

### Cursor streaming pour grands volumes

```typescript
async function processLargeDataset() {
    const collection = db.collection<UserDocument>('users');

    // ❌ MAUVAIS - Charge tout en mémoire
    // const users = await collection.find({}).toArray();

    // ✅ BON - Streaming avec cursor
    const cursor = collection.find({}).batchSize(100);

    let processedCount = 0;

    for await (const user of cursor) {
        // Traiter chaque document individuellement
        await processUser(user);
        processedCount++;

        if (processedCount % 1000 === 0) {
            console.log(`Processed ${processedCount} users`);
        }
    }

    console.log(`✅ Total processed: ${processedCount}`);
}
```

### Bulk operations

```typescript
async function bulkUpdateProducts(updates: Array<{
    productId: ObjectId;
    price: number;
}>) {
    const collection = db.collection<ProductDocument>('products');

    // Créer les opérations bulk
    const bulkOps = updates.map(update => ({
        updateOne: {
            filter: { _id: update.productId },
            update: {
                $set: {
                    'price.amount': update.price,
                    updatedAt: new Date()
                }
            }
        }
    }));

    // Exécuter en bulk (beaucoup plus performant)
    const result = await collection.bulkWrite(bulkOps, {
        ordered: false // Continue même si certaines échouent
    });

    console.log(`Updated: ${result.modifiedCount}/${updates.length}`);

    return result;
}
```

## Monitoring et logging

### Event listeners pour monitoring

```typescript
import { MongoClient } from 'mongodb';

const client = new MongoClient(uri, {
    monitorCommands: true
});

// Connection events
client.on('open', () => {
    console.log('📂 MongoDB connection opened');
});

client.on('close', () => {
    console.log('📪 MongoDB connection closed');
});

client.on('error', (error) => {
    console.error('❌ MongoDB connection error:', error);
});

// Command monitoring
client.on('commandStarted', (event) => {
    console.log(`🔵 Command ${event.commandName} started`);
});

client.on('commandSucceeded', (event) => {
    console.log(`✅ Command ${event.commandName} succeeded in ${event.duration}ms`);
});

client.on('commandFailed', (event) => {
    console.error(`❌ Command ${event.commandName} failed:`, event.failure);
});

// Connection pool events
client.on('connectionPoolCreated', () => {
    console.log('🏊 Connection pool created');
});

client.on('connectionCreated', () => {
    console.log('🔗 New connection created');
});

client.on('connectionClosed', () => {
    console.log('🔌 Connection closed');
});
```

## Bonnes pratiques récapitulatives

### ✅ DO - À faire

```typescript
// 1. Utiliser un singleton pour le client
const client = new MongoClient(uri);
await client.connect();

// 2. Typer vos documents avec TypeScript
interface User {
    _id: ObjectId;
    name: string;
    email: string;
}
const users = db.collection<User>('users');

// 3. Utiliser des projections
const user = await users.findOne(
    { email },
    { projection: { password: 0 } }
);

// 4. Fermer proprement les ressources
process.on('SIGINT', async () => {
    await client.close();
    process.exit(0);
});

// 5. Utiliser des index appropriés
await collection.createIndex({ email: 1 }, { unique: true });

// 6. Gérer les erreurs spécifiques
try {
    await collection.insertOne(doc);
} catch (error: any) {
    if (error.code === 11000) {
        // Handle duplicate key
    }
}

// 7. Utiliser le repository pattern
class UserRepository extends BaseRepository<User> {}

// 8. Valider les données avant insertion
const validated = userSchema.parse(userData);
```

### ❌ DON'T - À éviter

```typescript
// 1. ❌ Créer un nouveau client pour chaque requête
// 2. ❌ Ne pas typer vos collections
// 3. ❌ Charger des documents entiers inutilement
// 4. ❌ Oublier de fermer les connexions
// 5. ❌ Ignorer les erreurs
// 6. ❌ Faire des requêtes N+1
// 7. ❌ Utiliser .toArray() sur de gros datasets
// 8. ❌ Exposer les ObjectId raw dans les APIs
```

## Conclusion

Le driver Node.js MongoDB offre une API puissante et flexible pour interagir avec MongoDB. Les points clés à retenir :

1. **Architecture** : Utilisez le pattern singleton pour le client
2. **TypeScript** : Exploitez le typage fort pour la sécurité
3. **Patterns** : Adoptez le repository pattern pour la maintenabilité
4. **Performance** : Utilisez le pooling, les index, et les bulk operations
5. **Robustesse** : Gérez les erreurs, utilisez les transactions quand nécessaire
6. **Monitoring** : Surveillez vos opérations et performances

---


⏭️ [Driver Python (PyMongo)](/15-drivers-integration-applicative/03-driver-python-pymongo.md)
