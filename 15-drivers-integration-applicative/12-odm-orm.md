🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.12 ODM et ORM

## Introduction

Les **ODM** (Object-Document Mapper) et **ORM** (Object-Relational Mapper) sont des bibliothèques qui facilitent l'interaction entre le code applicatif orienté objet et les bases de données. Dans le contexte MongoDB, on parle principalement d'**ODM** puisque MongoDB est une base de données orientée document.

### Définitions

**ODM (Object-Document Mapper)**
- Traduit les objets du langage de programmation en documents MongoDB (BSON)
- Fournit une abstraction de haut niveau pour les opérations CRUD
- Gère la validation, les relations et les hooks de cycle de vie

**ORM (Object-Relational Mapper)**
- Conçu pour les bases de données relationnelles (SQL)
- Mappe les objets aux tables et lignes
- Gère les jointures et les relations complexes

> **Note importante** : Bien que certains outils se nomment "ORM" dans l'écosystème MongoDB (par convention ou par extension), ils sont techniquement des ODM.

---

## Pourquoi utiliser un ODM ?

### Avantages

#### 1. **Abstraction et productivité**

Les ODM permettent de manipuler des objets natifs du langage plutôt que des documents bruts :

```javascript
// Sans ODM (driver natif)
const user = await db.collection('users').findOne({ email: 'user@example.com' });
user.lastLogin = new Date();
await db.collection('users').updateOne(
  { _id: user._id },
  { $set: { lastLogin: user.lastLogin } }
);

// Avec ODM (Mongoose)
const user = await User.findOne({ email: 'user@example.com' });
user.lastLogin = new Date();
await user.save();
```

#### 2. **Validation des schémas**

Les ODM offrent une validation robuste avant l'insertion :

```python
# Avec PyMongo (sans ODM)
from pymongo import MongoClient

client = MongoClient()
db = client.mydb

# Pas de validation automatique
db.users.insert_one({
    "email": "invalid-email",  # Pas validé
    "age": "twenty"  # Type incorrect
})

# Avec MongoEngine (ODM)
from mongoengine import Document, StringField, IntField, EmailField

class User(Document):
    email = EmailField(required=True)
    age = IntField(min_value=0, max_value=150)

# Validation automatique
user = User(email="invalid-email", age="twenty")
user.save()  # Lève ValidationError
```

#### 3. **Relations et population**

Gestion simplifiée des références entre documents :

```javascript
// Mongoose - Définition avec références
const PostSchema = new Schema({
  title: String,
  author: { type: Schema.Types.ObjectId, ref: 'User' },
  comments: [{ type: Schema.Types.ObjectId, ref: 'Comment' }]
});

// Population automatique
const post = await Post.findById(postId)
  .populate('author')
  .populate('comments');

console.log(post.author.name); // Accès direct
```

#### 4. **Hooks et middleware**

Exécution de logique métier à des moments clés :

```java
// Spring Data MongoDB
@Document(collection = "users")
public class User {
    @Id
    private String id;
    private String email;
    private Date createdAt;

    @PrePersist
    public void prePersist() {
        this.createdAt = new Date();
        this.email = this.email.toLowerCase();
    }
}
```

#### 5. **Requêtes typées**

Sécurité au moment de la compilation :

```csharp
// C# avec MongoDB.Driver + LINQ
var recentUsers = await users
    .AsQueryable()
    .Where(u => u.CreatedAt > DateTime.Now.AddDays(-7))
    .OrderByDescending(u => u.CreatedAt)
    .Take(10)
    .ToListAsync();
```

### Inconvénients

#### 1. **Surcharge de performance**

```javascript
// Comparaison de performance

// Driver natif - Rapide
const start = Date.now();
const users = await db.collection('users').find({ active: true }).toArray();
console.log(`Native: ${Date.now() - start}ms`); // ~10ms

// ODM - Plus lent (hydratation, validation)
const start2 = Date.now();
const usersODM = await User.find({ active: true });
console.log(`ODM: ${Date.now() - start2}ms`); // ~25ms
```

#### 2. **Courbe d'apprentissage**

Nécessite d'apprendre l'API de l'ODM en plus de MongoDB :

```python
# Deux syntaxes à maîtriser

# Syntaxe MongoDB native
db.users.find({"age": {"$gte": 18, "$lte": 30}})

# Syntaxe MongoEngine
User.objects(age__gte=18, age__lte=30)

# Requêtes complexes peuvent être limitées
```

#### 3. **Abstraction excessive**

Peut masquer les opérations MongoDB réelles :

```javascript
// Ce qui se passe vraiment ?
await User.find().populate('posts').populate('followers');

// Génère potentiellement plusieurs requêtes
// 1. find users
// 2. find posts where author in [user_ids]
// 3. find users where _id in [follower_ids]
// Risque de N+1 queries
```

#### 4. **Flexibilité réduite**

Les requêtes très spécifiques peuvent être difficiles :

```go
// Sans ORM - Agrégation complexe facile
pipeline := bson.A{
    bson.D{{"$match", bson.D{{"status", "active"}}}},
    bson.D{{"$lookup", bson.D{
        {"from", "orders"},
        {"localField", "_id"},
        {"foreignField", "userId"},
        {"as", "orders"},
    }}},
    bson.D{{"$addFields", bson.D{
        {"totalSpent", bson.D{{"$sum", "$orders.amount"}}},
    }}},
}

// Avec certains ORM - Plus verbeux ou limité
```

---

## Panorama des ODM principaux

### Écosystème Node.js

| ODM | Description | Points forts |
|-----|-------------|--------------|
| **Mongoose** | L'ODM le plus populaire | Validation riche, middleware, plugins, grande communauté |
| **Typegoose** | Mongoose + TypeScript | Type-safety, décorateurs, génération de schémas |
| **Prisma** | ORM moderne multi-DB | Type-safety, migrations, introspection, génération de types |
| **TypeORM** | ORM multi-DB avec support MongoDB | Décorateurs, Active Record/Data Mapper, relations |

### Écosystème Python

| ODM | Description | Points forts |
|-----|-------------|--------------|
| **MongoEngine** | ODM Django-like | Validation, requêtes ORM-style, documents imbriqués |
| **Motor** | Async pour Tornado/asyncio | Performance asynchrone, compatible PyMongo |
| **Beanie** | ODM moderne async avec Pydantic | FastAPI integration, validation Pydantic, type hints |
| **ODMantic** | ODM basé sur Pydantic | Type-safe, validation moderne, async natif |

### Écosystème Java

| Framework | Description | Points forts |
|-----------|-------------|--------------|
| **Spring Data MongoDB** | Intégration Spring | Repositories, annotations, requêtes dérivées |
| **Morphia** | ODM léger | Simple, annotations, type-safe |
| **Jongo** | Query-as-you-write | Syntaxe proche du shell MongoDB |

### Écosystème .NET

| ODM | Description | Points forts |
|-----|-------------|--------------|
| **MongoDB.Entities** | ODM moderne .NET | Fluent API, relations, transactions |
| **MongoFramework** | Entity Framework-like | Conventions EF, change tracking |
| **Driver officiel + LINQ** | Solution Microsoft | LINQ natif, POCOs, performance |

### Autres langages

| Langage | ODM/Framework | Notes |
|---------|---------------|-------|
| **PHP** | Doctrine ODM | Intégration Symfony, annotations |
| **Ruby** | Mongoid | Rails-like, validations ActiveModel |
| **Go** | mongo-go-driver + structs | Approche minimaliste avec tags struct |
| **Rust** | mongodb + serde | Sérialisation via serde, type-safe |

---

## Comparaison approfondie : Driver natif vs ODM

### Exemple 1 : Création d'un utilisateur

#### Driver natif (Node.js)

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(uri);
await client.connect();
const db = client.db('myapp');

// Validation manuelle
function validateUser(userData) {
  if (!userData.email || !userData.email.includes('@')) {
    throw new Error('Invalid email');
  }
  if (!userData.age || userData.age < 0) {
    throw new Error('Invalid age');
  }
}

const userData = {
  email: 'john@example.com',
  name: 'John Doe',
  age: 30,
  createdAt: new Date(),
  updatedAt: new Date()
};

validateUser(userData);

const result = await db.collection('users').insertOne(userData);
console.log(`User created with id: ${result.insertedId}`);
```

#### ODM (Mongoose)

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    validate: {
      validator: (v) => /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/.test(v),
      message: 'Invalid email format'
    }
  },
  name: {
    type: String,
    required: true,
    trim: true,
    minlength: 2,
    maxlength: 100
  },
  age: {
    type: Number,
    required: true,
    min: 0,
    max: 150
  }
}, {
  timestamps: true // createdAt, updatedAt automatiques
});

// Middleware
userSchema.pre('save', function(next) {
  console.log(`Saving user: ${this.email}`);
  next();
});

const User = mongoose.model('User', userSchema);

// Utilisation
const user = new User({
  email: 'john@example.com',
  name: 'John Doe',
  age: 30
});

await user.save(); // Validation + middleware automatiques
console.log(`User created with id: ${user._id}`);
```

### Exemple 2 : Requêtes complexes

#### Python - PyMongo vs MongoEngine

```python
from datetime import datetime, timedelta
from pymongo import MongoClient

# PyMongo - Driver natif
client = MongoClient()
db = client.myapp

# Requête complexe
recent_active_users = list(db.users.find({
    'status': 'active',
    'lastLogin': {'$gte': datetime.now() - timedelta(days=7)},
    'subscription.plan': {'$in': ['premium', 'enterprise']}
}).sort('lastLogin', -1).limit(10))

for user in recent_active_users:
    print(f"User: {user['email']}, Last login: {user['lastLogin']}")
```

```python
from mongoengine import Document, StringField, DateTimeField, EmbeddedDocument, EmbeddedDocumentField
from datetime import datetime, timedelta

# MongoEngine - ODM
class Subscription(EmbeddedDocument):
    plan = StringField(choices=['free', 'premium', 'enterprise'])
    startDate = DateTimeField()

class User(Document):
    email = StringField(required=True, unique=True)
    status = StringField(choices=['active', 'inactive'])
    lastLogin = DateTimeField()
    subscription = EmbeddedDocumentField(Subscription)

    meta = {
        'collection': 'users',
        'indexes': ['lastLogin', 'status']
    }

# Requête avec ORM-style
seven_days_ago = datetime.now() - timedelta(days=7)
recent_active_users = User.objects(
    status='active',
    lastLogin__gte=seven_days_ago,
    subscription__plan__in=['premium', 'enterprise']
).order_by('-lastLogin').limit(10)

for user in recent_active_users:
    print(f"User: {user.email}, Last login: {user.lastLogin}")
    print(f"Plan: {user.subscription.plan}")
```

### Exemple 3 : Relations et population

#### Java - Driver natif vs Spring Data

```java
// Driver natif MongoDB Java
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import org.bson.Document;
import org.bson.types.ObjectId;

MongoClient mongoClient = MongoClients.create("mongodb://localhost:27017");
MongoDatabase database = mongoClient.getDatabase("myapp");

// Récupérer un post
MongoCollection<Document> posts = database.getCollection("posts");
Document post = posts.find(eq("_id", new ObjectId(postId))).first();

// Récupérer l'auteur manuellement
ObjectId authorId = post.getObjectId("authorId");
MongoCollection<Document> users = database.getCollection("users");
Document author = users.find(eq("_id", authorId)).first();

System.out.println("Post: " + post.getString("title"));
System.out.println("Author: " + author.getString("name"));
```

```java
// Spring Data MongoDB
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.DBRef;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.repository.MongoRepository;

@Document(collection = "posts")
public class Post {
    @Id
    private String id;
    private String title;
    private String content;

    @DBRef
    private User author;

    // Getters, setters
}

@Document(collection = "users")
public class User {
    @Id
    private String id;
    private String name;
    private String email;

    // Getters, setters
}

// Repository
public interface PostRepository extends MongoRepository<Post, String> {
    List<Post> findByAuthor(User author);
}

// Utilisation
Post post = postRepository.findById(postId).orElseThrow();
System.out.println("Post: " + post.getTitle());
System.out.println("Author: " + post.getAuthor().getName()); // Population automatique
```

---

## Quand utiliser un ODM ?

### ✅ Utilisez un ODM si :

1. **Projet applicatif standard**
   - CRUD classique avec objets métier
   - Relations entre entités bien définies
   - Validation de schéma importante

2. **Équipe habituée aux ORM**
   - Migration depuis SQL (Rails, Django, Entity Framework)
   - Patterns familiers (Active Record, Repository)

3. **Besoin de productivité**
   - Prototypage rapide
   - Startups en phase MVP
   - Applications CRUD-intensives

4. **Type-safety critique**
   - TypeScript, Java, C#
   - Contrats d'API stricts
   - Grandes équipes

### ❌ Évitez un ODM si :

1. **Performance critique**
   - Microservices haute fréquence
   - Traitement de gros volumes
   - Latence ultra-faible requise

2. **Requêtes très spécifiques**
   - Agrégations complexes fréquentes
   - Pipelines MongoDB avancés
   - Analytics en temps réel

3. **Schéma très dynamique**
   - Documents polymorphes
   - Schéma évolutif en production
   - Données semi-structurées

4. **Contrôle total nécessaire**
   - Optimisations fines
   - Debugging approfondi
   - Opérations MongoDB natives

---

## Bonnes pratiques avec les ODM

### 1. Ne pas abuser de la population

```javascript
// ❌ Mauvais : Population en cascade
const post = await Post.findById(id)
  .populate('author')
  .populate('comments')
  .populate('comments.author')
  .populate('tags');
// Génère 4+ requêtes !

// ✅ Bon : Agrégation si nécessaire
const post = await Post.aggregate([
  { $match: { _id: mongoose.Types.ObjectId(id) } },
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
    $lookup: {
      from: 'comments',
      let: { postId: '$_id' },
      pipeline: [
        { $match: { $expr: { $eq: ['$postId', '$$postId'] } } },
        { $limit: 10 },
        {
          $lookup: {
            from: 'users',
            localField: 'authorId',
            foreignField: '_id',
            as: 'author'
          }
        },
        { $unwind: '$author' }
      ],
      as: 'comments'
    }
  }
]);
```

### 2. Utiliser lean() pour la lecture seule

```javascript
// ❌ Instanciation complète non nécessaire
const users = await User.find({ active: true });
// Retourne des documents Mongoose avec tous les methods

// ✅ Lean pour lecture pure
const users = await User.find({ active: true }).lean();
// Retourne des objets JavaScript simples (2-5x plus rapide)

// Parfait pour les APIs read-only
app.get('/api/users', async (req, res) => {
  const users = await User.find().select('name email').lean();
  res.json(users);
});
```

### 3. Indexes définis dans le schéma

```python
# MongoEngine - Indexes dans la classe
class Article(Document):
    title = StringField(required=True)
    slug = StringField(unique=True)
    author = ReferenceField(User)
    published_at = DateTimeField()
    status = StringField(choices=['draft', 'published'])
    tags = ListField(StringField())

    meta = {
        'collection': 'articles',
        'indexes': [
            'slug',  # Index simple
            'author',
            '-published_at',  # Index descendant
            {
                'fields': ['status', '-published_at'],
                'name': 'status_published_idx'
            },
            {
                'fields': ['tags'],
                'sparse': True
            },
            {
                'fields': ['$title', '$content'],  # Text index
                'default_language': 'english'
            }
        ]
    }
```

### 4. Validation multi-niveaux

```typescript
// TypeScript + Typegoose
import { prop, getModelForClass, pre } from '@typegoose/typegoose';
import validator from 'validator';

@pre<User>('save', function() {
  // Logique métier avant sauvegarde
  if (this.isModified('email')) {
    this.emailVerified = false;
  }
})
class User {
  @prop({
    required: true,
    unique: true,
    lowercase: true,
    validate: {
      validator: (v: string) => validator.isEmail(v),
      message: 'Invalid email format'
    }
  })
  public email!: string;

  @prop({
    required: true,
    minlength: 8,
    validate: {
      validator: function(v: string) {
        return /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(v);
      },
      message: 'Password must contain uppercase, lowercase, and number'
    }
  })
  public password!: string;

  @prop({ default: false })
  public emailVerified?: boolean;

  @prop({ type: () => [String], default: [] })
  public roles!: string[];

  // Méthode d'instance
  public hasRole(role: string): boolean {
    return this.roles.includes(role);
  }

  // Méthode statique
  public static async findByEmail(email: string) {
    return this.findOne({ email: email.toLowerCase() });
  }
}

const UserModel = getModelForClass(User);
```

### 5. Éviter les transformations coûteuses

```javascript
// ❌ Mauvais : Transformation dans un getter
userSchema.virtual('fullName').get(function() {
  return this.firstName + ' ' + this.lastName;
});

// Si utilisé dans une liste de 1000 users, calcul 1000 fois

// ✅ Bon : Champ calculé à l'insertion/update
userSchema.pre('save', function(next) {
  this.fullName = `${this.firstName} ${this.lastName}`;
  next();
});

// Ou via agrégation si besoin ponctuel
const users = await User.aggregate([
  {
    $addFields: {
      fullName: { $concat: ['$firstName', ' ', '$lastName'] }
    }
  }
]);
```

### 6. Gestion des transactions avec ODM

```javascript
// Mongoose + Transactions
const session = await mongoose.startSession();
session.startTransaction();

try {
  // Créer une commande
  const order = new Order({
    userId: userId,
    items: cartItems,
    total: calculateTotal(cartItems)
  });
  await order.save({ session });

  // Décrémenter le stock
  for (const item of cartItems) {
    await Product.findByIdAndUpdate(
      item.productId,
      { $inc: { stock: -item.quantity } },
      { session }
    );
  }

  // Vider le panier
  await Cart.findOneAndUpdate(
    { userId: userId },
    { $set: { items: [] } },
    { session }
  );

  await session.commitTransaction();
  console.log('Order placed successfully');
} catch (error) {
  await session.abortTransaction();
  console.error('Transaction failed:', error);
  throw error;
} finally {
  session.endSession();
}
```

### 7. Pagination efficace

```python
# MongoEngine - Pagination cursor-based
from mongoengine import Document, StringField, DateTimeField, queryset_manager

class Post(Document):
    title = StringField(required=True)
    created_at = DateTimeField(required=True)

    @queryset_manager
    def paginate_cursor(cls, queryset, last_id=None, limit=20):
        """Pagination basée sur curseur (plus efficace que skip)"""
        if last_id:
            queryset = queryset.filter(id__lt=last_id)
        return queryset.order_by('-created_at').limit(limit)

# Utilisation
page1 = Post.paginate_cursor(limit=20)
last_post_id = page1[-1].id if page1 else None

page2 = Post.paginate_cursor(last_id=last_post_id, limit=20)
```

### 8. Projection pour économiser la bande passante

```java
// Spring Data MongoDB - Projection
public interface UserProjection {
    String getId();
    String getName();
    String getEmail();
    // Pas de password, créatedAt, etc.
}

public interface UserRepository extends MongoRepository<User, String> {
    List<UserProjection> findAllProjectedBy();

    @Query(value = "{}", fields = "{ 'name': 1, 'email': 1 }")
    List<User> findAllMinimal();
}

// Utilisation
List<UserProjection> users = userRepository.findAllProjectedBy();
// Récupère uniquement les champs nécessaires
```

### 9. Hooks pour la logique métier

```javascript
// Mongoose - Middleware pour audit trail
const productSchema = new mongoose.Schema({
  name: String,
  price: Number,
  stock: Number,
  history: [{
    action: String,
    user: mongoose.Schema.Types.ObjectId,
    timestamp: Date,
    changes: mongoose.Schema.Types.Mixed
  }]
});

productSchema.pre('save', function(next) {
  if (this.isModified()) {
    const changes = this.modifiedPaths().reduce((acc, path) => {
      acc[path] = {
        old: this._doc[path],
        new: this[path]
      };
      return acc;
    }, {});

    this.history.push({
      action: this.isNew ? 'create' : 'update',
      user: this._currentUser, // Contexte applicatif
      timestamp: new Date(),
      changes
    });
  }
  next();
});

// Utilisation
product._currentUser = req.user._id; // Injecter le contexte
product.price = 29.99;
await product.save(); // Historique enregistré automatiquement
```

### 10. Tests avec ODM

```javascript
// Jest + Mongoose - Tests d'intégration
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

describe('User Model', () => {
  let mongoServer;

  beforeAll(async () => {
    mongoServer = await MongoMemoryServer.create();
    await mongoose.connect(mongoServer.getUri());
  });

  afterAll(async () => {
    await mongoose.disconnect();
    await mongoServer.stop();
  });

  afterEach(async () => {
    await User.deleteMany({});
  });

  it('should create a valid user', async () => {
    const userData = {
      email: 'test@example.com',
      name: 'Test User',
      age: 25
    };

    const user = new User(userData);
    const savedUser = await user.save();

    expect(savedUser._id).toBeDefined();
    expect(savedUser.email).toBe(userData.email);
    expect(savedUser.createdAt).toBeInstanceOf(Date);
  });

  it('should not create user with invalid email', async () => {
    const user = new User({
      email: 'invalid-email',
      name: 'Test User',
      age: 25
    });

    await expect(user.save()).rejects.toThrow();
  });

  it('should enforce unique email constraint', async () => {
    const userData = {
      email: 'duplicate@example.com',
      name: 'User One',
      age: 25
    };

    await User.create(userData);

    const duplicate = new User({
      email: 'duplicate@example.com',
      name: 'User Two',
      age: 30
    });

    await expect(duplicate.save()).rejects.toThrow(/duplicate key/);
  });
});
```

---

## Choix d'un ODM : Critères de décision

### Matrice de décision

| Critère | Questions à se poser | Poids |
|---------|---------------------|-------|
| **Maturité** | Communauté active ? Documentation complète ? | ⭐⭐⭐ |
| **Performance** | Overhead acceptable ? Benchmarks disponibles ? | ⭐⭐⭐ |
| **Type-safety** | Support TypeScript/types fort ? Validation compile-time ? | ⭐⭐ |
| **Fonctionnalités** | Relations ? Transactions ? Hooks ? Migrations ? | ⭐⭐⭐ |
| **Courbe d'apprentissage** | Complexité ? Documentation ? Exemples ? | ⭐⭐ |
| **Maintenance** | Dernier commit ? Réactivité aux issues ? | ⭐⭐⭐ |
| **Intégration** | Framework utilisé ? Plugins disponibles ? | ⭐⭐ |

### Recommandations par contexte

```
Node.js + REST API → Mongoose (maturité, communauté)
Node.js + TypeScript → Typegoose ou Prisma
Python + Django/Flask → MongoEngine (familiarité)
Python + FastAPI → Beanie (async natif, Pydantic)
Java + Spring → Spring Data MongoDB (intégration)
.NET + ASP.NET Core → MongoDB.Entities ou driver + LINQ
Performance critique → Driver natif + structs/POCOs
```

---

## Patterns d'architecture avec ODM

### Pattern Repository

```typescript
// Séparation des préoccupations
// user.model.ts
import { prop, getModelForClass } from '@typegoose/typegoose';

export class User {
  @prop({ required: true, unique: true })
  public email!: string;

  @prop({ required: true })
  public name!: string;

  @prop({ default: true })
  public active!: boolean;
}

export const UserModel = getModelForClass(User);

// user.repository.ts
export class UserRepository {
  async findById(id: string): Promise<User | null> {
    return UserModel.findById(id).lean();
  }

  async findByEmail(email: string): Promise<User | null> {
    return UserModel.findOne({ email: email.toLowerCase() }).lean();
  }

  async create(userData: Partial<User>): Promise<User> {
    const user = new UserModel(userData);
    return user.save();
  }

  async update(id: string, updates: Partial<User>): Promise<User | null> {
    return UserModel.findByIdAndUpdate(
      id,
      { $set: updates },
      { new: true, runValidators: true }
    ).lean();
  }

  async delete(id: string): Promise<boolean> {
    const result = await UserModel.deleteOne({ _id: id });
    return result.deletedCount > 0;
  }

  async findActive(limit: number = 50): Promise<User[]> {
    return UserModel.find({ active: true })
      .sort({ createdAt: -1 })
      .limit(limit)
      .lean();
  }
}

// user.service.ts
export class UserService {
  constructor(private userRepo: UserRepository) {}

  async registerUser(email: string, name: string): Promise<User> {
    const existing = await this.userRepo.findByEmail(email);
    if (existing) {
      throw new Error('Email already registered');
    }

    return this.userRepo.create({ email, name });
  }

  async deactivateUser(id: string): Promise<void> {
    const user = await this.userRepo.update(id, { active: false });
    if (!user) {
      throw new Error('User not found');
    }
  }
}
```

### Pattern Unit of Work (Transactions)

```python
# unit_of_work.py
from mongoengine import connect
from contextlib import contextmanager

class UnitOfWork:
    def __init__(self):
        self.client = connect('myapp').get_connection()

    @contextmanager
    def transaction(self):
        """Context manager pour transactions"""
        with self.client.start_session() as session:
            with session.start_transaction():
                try:
                    yield session
                except Exception as e:
                    session.abort_transaction()
                    raise e

# order.service.py
class OrderService:
    def __init__(self, uow: UnitOfWork):
        self.uow = uow

    def create_order(self, user_id, items):
        with self.uow.transaction() as session:
            # Vérifier le stock
            for item in items:
                product = Product.objects(id=item['product_id']).first()
                if product.stock < item['quantity']:
                    raise ValueError(f"Insufficient stock for {product.name}")

            # Créer la commande
            order = Order(
                user_id=user_id,
                items=items,
                total=self._calculate_total(items)
            )
            order.save(session=session)

            # Décrémenter le stock
            for item in items:
                Product.objects(id=item['product_id']).update_one(
                    dec__stock=item['quantity'],
                    session=session
                )

            return order
```

---

## Migration progressive vers un ODM

### Étape 1 : Coexistence

```javascript
// Garder le driver natif pour les requêtes complexes
const { MongoClient } = require('mongodb');
const mongoose = require('mongoose');

// Connexion partagée
const uri = 'mongodb://localhost:27017/myapp';

// Driver natif pour analytics
const client = new MongoClient(uri);
await client.connect();
const nativeDb = client.db();

// Mongoose pour CRUD applicatif
await mongoose.connect(uri);

// Utilisation mixte
class AnalyticsService {
  async getDailyStats() {
    // Agrégation complexe avec driver natif
    return nativeDb.collection('events').aggregate([
      {
        $group: {
          _id: {
            $dateToString: { format: '%Y-%m-%d', date: '$timestamp' }
          },
          count: { $sum: 1 }
        }
      }
    ]).toArray();
  }
}

class UserService {
  async createUser(userData) {
    // CRUD simple avec Mongoose
    const user = new User(userData);
    return user.save();
  }
}
```

### Étape 2 : Migration progressive

```javascript
// Phase 1 : Lire avec les deux, écrire avec ODM
async function getUserById(id) {
  // Lire d'abord avec ODM
  let user = await User.findById(id);

  if (!user) {
    // Fallback sur driver natif si pas encore migré
    const doc = await db.collection('users').findOne({ _id: ObjectId(id) });
    if (doc) {
      // Optionnel : migrer le document
      user = new User(doc);
      await user.save();
    }
  }

  return user;
}

// Phase 2 : Tout avec ODM
async function getUserById(id) {
  return User.findById(id);
}
```

---

## Conclusion

Les ODM offrent un **compromis entre productivité et performance**. Ils sont particulièrement adaptés aux applications CRUD standards avec des schémas stables et des relations bien définies.

### Points clés à retenir

1. **Évaluer le besoin réel** : Tous les projets n'ont pas besoin d'un ODM
2. **Comprendre le coût** : Overhead de 10-40% sur les opérations
3. **Utiliser judicieusement** : ODM pour le CRUD, driver natif pour les agrégations
4. **Connaître les limites** : Ne pas tout abstraire
5. **Tester les performances** : Benchmarker sur votre workload réel

### Prochaines sections

Les sections suivantes détaillent les ODM principaux :
- **15.12.1** : Mongoose (Node.js) - L'ODM le plus populaire
- **15.12.2** : Motor (Python async) - Pour les applications asynchrones
- **15.12.3** : Spring Data MongoDB (Java) - Intégration Spring

---

**Ressources complémentaires**

- [Mongoose Documentation](https://mongoosejs.com/)
- [MongoEngine Documentation](http://mongoengine.org/)
- [Spring Data MongoDB Reference](https://spring.io/projects/spring-data-mongodb)
- [MongoDB Best Practices](https://www.mongodb.com/docs/manual/applications/data-models/)

⏭️ [Mongoose (Node.js)](/15-drivers-integration-applicative/12.1-mongoose-nodejs.md)
