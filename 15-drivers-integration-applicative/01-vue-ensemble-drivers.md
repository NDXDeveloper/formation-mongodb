🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.1 Vue d'ensemble des drivers officiels

## Introduction

MongoDB propose des **drivers officiels** pour tous les langages de programmation majeurs. Ces drivers sont maintenus par MongoDB Inc. et garantissent une compatibilité optimale avec les dernières fonctionnalités du serveur MongoDB. Comprendre les spécificités de chaque driver est essentiel pour faire le bon choix technologique et optimiser vos applications.

## Architecture des drivers MongoDB

### Couches d'abstraction

Tous les drivers MongoDB suivent une architecture en couches similaire :

```
┌─────────────────────────────────────────────┐
│        API publique (idiomatique)           │
│  (Méthodes spécifiques au langage)          │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│        Couche d'abstraction CRUD            │
│  (find, insert, update, delete, etc.)       │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          Encodeur/Décodeur BSON             │
│  (Sérialisation des documents)              │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│      Gestionnaire de connexions             │
│  (Connection pooling, failover)             │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│        MongoDB Wire Protocol                │
│  (Communication réseau TCP/TLS)             │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
            MongoDB Server
```

### Spécifications communes

Tous les drivers officiels implémentent :

1. **MongoDB Wire Protocol** - Le protocole de communication standardisé
2. **BSON** - Sérialisation/désérialisation binaire
3. **Connection String URI** - Format standard de connexion
4. **Server Discovery and Monitoring (SDAM)** - Détection automatique de la topologie
5. **Connection Pooling** - Gestion efficace des connexions
6. **Automatic Retry** - Retry automatique pour certaines erreurs
7. **Causally Consistent Sessions** - Sessions cohérentes
8. **Transactions multi-documents** - Support des transactions ACID

## Catalogue des drivers officiels

### 1. Driver Node.js

**Langage** : JavaScript / TypeScript
**Package** : `mongodb`
**Repository** : https://github.com/mongodb/node-mongodb-native
**Documentation** : https://mongodb.github.io/node-mongodb-native/

#### Caractéristiques

```javascript
// Installation
npm install mongodb

// Version minimale Node.js : 14.20.1+
// Support TypeScript natif
```

**Points forts** :
- ✅ Async/await natif avec Promises
- ✅ Support TypeScript excellent (types inclus)
- ✅ Performance élevée grâce à V8
- ✅ Écosystème npm très riche
- ✅ Parfait pour les applications temps réel
- ✅ Connection pooling automatique
- ✅ Support des Streams Node.js

**Points d'attention** :
- ⚠️ Typage dynamique JavaScript (résolu avec TypeScript)
- ⚠️ Gestion mémoire à surveiller (garbage collector)

#### Exemple d'initialisation

```javascript
// JavaScript moderne (ES modules)
import { MongoClient, ObjectId } from 'mongodb';

const uri = "mongodb://localhost:27017";
const client = new MongoClient(uri, {
    maxPoolSize: 50,
    minPoolSize: 10,
    maxIdleTimeMS: 30000,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    family: 4, // IPv4 uniquement
    retryWrites: true,
    retryReads: true
});

async function main() {
    try {
        await client.connect();
        console.log('✅ Connecté à MongoDB');

        const db = client.db('myapp');
        const users = db.collection('users');

        // Opération exemple
        const result = await users.insertOne({
            name: "Alice",
            email: "alice@example.com",
            createdAt: new Date()
        });

        console.log(`Document inséré : ${result.insertedId}`);

    } catch (error) {
        console.error('❌ Erreur:', error);
    } finally {
        await client.close();
    }
}

main();
```

```typescript
// TypeScript avec typage fort
import { MongoClient, Db, Collection, ObjectId } from 'mongodb';

interface User {
    _id?: ObjectId;
    name: string;
    email: string;
    age?: number;
    createdAt: Date;
}

class DatabaseService {
    private client: MongoClient;
    private db!: Db;

    constructor(uri: string) {
        this.client = new MongoClient(uri, {
            maxPoolSize: 50,
            minPoolSize: 10
        });
    }

    async connect(): Promise<void> {
        await this.client.connect();
        this.db = this.client.db('myapp');
        console.log('✅ Connecté à MongoDB');
    }

    getUsersCollection(): Collection<User> {
        return this.db.collection<User>('users');
    }

    async close(): Promise<void> {
        await this.client.close();
    }
}

// Utilisation
const dbService = new DatabaseService('mongodb://localhost:27017');
await dbService.connect();

const users = dbService.getUsersCollection();
const user = await users.findOne({ email: "alice@example.com" });
// TypeScript connaît le type de 'user' : User | null
```

**Versions et compatibilité** :
- Driver 6.x : MongoDB 3.6+, Node.js 16.20.1+
- Driver 5.x : MongoDB 3.6+, Node.js 14.20.1+
- Driver 4.x : MongoDB 2.6+, Node.js 12+

---

### 2. Driver Python (PyMongo)

**Langage** : Python
**Package** : `pymongo`
**Repository** : https://github.com/mongodb/mongo-python-driver
**Documentation** : https://pymongo.readthedocs.io/

#### Caractéristiques

```python
# Installation
pip install pymongo

# Version minimale Python : 3.7+
# Pour le support async : pip install motor
```

**Points forts** :
- ✅ API pythonique et intuitive
- ✅ Excellent pour le data science et machine learning
- ✅ Integration parfaite avec NumPy, Pandas
- ✅ Motor pour les applications async (Tornado, asyncio)
- ✅ Support BSON natif
- ✅ Documentation exhaustive

**Points d'attention** :
- ⚠️ Pas de typage statique (résolu avec type hints Python 3.5+)
- ⚠️ Version synchrone par défaut (utiliser Motor pour async)
- ⚠️ Performance moindre que Java/Go pour haute concurrence

#### Exemple d'initialisation

```python
# PyMongo synchrone
from pymongo import MongoClient, ASCENDING, DESCENDING
from pymongo.errors import ConnectionFailure, OperationFailure
from datetime import datetime
from bson.objectid import ObjectId

# Configuration du client
client = MongoClient(
    'mongodb://localhost:27017/',
    maxPoolSize=50,
    minPoolSize=10,
    maxIdleTimeMS=30000,
    serverSelectionTimeoutMS=5000,
    socketTimeoutMS=45000,
    retryWrites=True,
    retryReads=True
)

try:
    # Vérifier la connexion
    client.admin.command('ping')
    print('✅ Connecté à MongoDB')

    # Accès à la base et collection
    db = client['myapp']
    users = db['users']

    # Opération exemple
    result = users.insert_one({
        'name': 'Alice',
        'email': 'alice@example.com',
        'createdAt': datetime.now()
    })

    print(f'Document inséré : {result.inserted_id}')

except ConnectionFailure as e:
    print(f'❌ Erreur de connexion: {e}')
finally:
    client.close()
```

```python
# Motor pour applications asynchrones
import asyncio
from motor.motor_asyncio import AsyncIOMotorClient
from datetime import datetime

async def main():
    # Client asynchrone
    client = AsyncIOMotorClient(
        'mongodb://localhost:27017/',
        maxPoolSize=50
    )

    try:
        # Vérifier la connexion
        await client.admin.command('ping')
        print('✅ Connecté à MongoDB (async)')

        db = client['myapp']
        users = db['users']

        # Opération asynchrone
        result = await users.insert_one({
            'name': 'Bob',
            'email': 'bob@example.com',
            'createdAt': datetime.now()
        })

        print(f'Document inséré : {result.inserted_id}')

        # Recherche asynchrone
        async for user in users.find({'name': 'Bob'}):
            print(f'Utilisateur trouvé : {user}')

    finally:
        client.close()

# Exécution
asyncio.run(main())
```

```python
# Avec type hints (Python 3.9+)
from typing import Optional, List, TypedDict
from pymongo import MongoClient
from bson import ObjectId

class User(TypedDict):
    _id: ObjectId
    name: str
    email: str
    age: Optional[int]

class UserRepository:
    def __init__(self, client: MongoClient):
        self.collection = client['myapp']['users']

    def find_by_email(self, email: str) -> Optional[User]:
        return self.collection.find_one({'email': email})

    def find_all_active(self) -> List[User]:
        return list(self.collection.find({'status': 'active'}))

    def create(self, user_data: dict) -> ObjectId:
        result = self.collection.insert_one(user_data)
        return result.inserted_id

# Utilisation
client = MongoClient('mongodb://localhost:27017/')
repo = UserRepository(client)
user = repo.find_by_email('alice@example.com')
```

**Versions et compatibilité** :
- PyMongo 4.x : MongoDB 3.6+, Python 3.7+
- PyMongo 3.x : MongoDB 2.6+, Python 2.7+/3.4+
- Motor 3.x : Python 3.7+, asyncio

---

### 3. Driver Java

**Langage** : Java
**Package** : `mongodb-driver-sync` / `mongodb-driver-reactivestreams`
**Repository** : https://github.com/mongodb/mongo-java-driver
**Documentation** : https://mongodb.github.io/mongo-java-driver/

#### Caractéristiques

```xml
<!-- Maven -->
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>5.1.0</version>
</dependency>

<!-- Pour reactive streams -->
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-reactivestreams</artifactId>
    <version>5.1.0</version>
</dependency>
```

```gradle
// Gradle
implementation 'org.mongodb:mongodb-driver-sync:5.1.0'
implementation 'org.mongodb:mongodb-driver-reactivestreams:5.1.0'
```

**Points forts** :
- ✅ Performance exceptionnelle
- ✅ Typage fort et sécurité à la compilation
- ✅ Excellent pour applications d'entreprise
- ✅ Support reactive streams (Project Reactor, RxJava)
- ✅ Intégration Spring Data MongoDB
- ✅ JMX monitoring intégré
- ✅ Mature et stable

**Points d'attention** :
- ⚠️ Verbosité du code (résolu avec Lombok/Kotlin)
- ⚠️ Courbe d'apprentissage plus élevée
- ⚠️ Empreinte mémoire JVM

#### Exemple d'initialisation

```java
// Driver synchrone
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.MongoCollection;
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.ServerApi;
import com.mongodb.ServerApiVersion;

import org.bson.Document;
import org.bson.codecs.configuration.CodecRegistry;
import org.bson.codecs.pojo.PojoCodecProvider;

import java.util.Date;
import java.util.concurrent.TimeUnit;

import static org.bson.codecs.configuration.CodecRegistries.fromProviders;
import static org.bson.codecs.configuration.CodecRegistries.fromRegistries;
import static com.mongodb.MongoClientSettings.getDefaultCodecRegistry;

public class MongoDBConnection {

    public static void main(String[] args) {
        // Configuration détaillée
        ConnectionString connectionString = new ConnectionString(
            "mongodb://localhost:27017"
        );

        // Codec registry pour POJO mapping
        CodecRegistry pojoCodecRegistry = fromRegistries(
            getDefaultCodecRegistry(),
            fromProviders(PojoCodecProvider.builder().automatic(true).build())
        );

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .codecRegistry(pojoCodecRegistry)
            .applyToConnectionPoolSettings(builder ->
                builder.maxSize(50)
                       .minSize(10)
                       .maxWaitTime(30, TimeUnit.SECONDS)
                       .maxConnectionIdleTime(30, TimeUnit.SECONDS)
            )
            .applyToSocketSettings(builder ->
                builder.connectTimeout(5, TimeUnit.SECONDS)
                       .readTimeout(45, TimeUnit.SECONDS)
            )
            .applyToServerSettings(builder ->
                builder.heartbeatFrequency(10, TimeUnit.SECONDS)
            )
            .retryWrites(true)
            .retryReads(true)
            .build();

        try (MongoClient mongoClient = MongoClients.create(settings)) {
            // Vérifier la connexion
            mongoClient.getDatabase("admin")
                       .runCommand(new Document("ping", 1));
            System.out.println("✅ Connecté à MongoDB");

            // Accès à la base et collection
            MongoDatabase database = mongoClient.getDatabase("myapp");
            MongoCollection<Document> collection = database.getCollection("users");

            // Opération exemple
            Document user = new Document("name", "Alice")
                .append("email", "alice@example.com")
                .append("createdAt", new Date());

            collection.insertOne(user);
            System.out.println("Document inséré : " + user.getObjectId("_id"));

        } catch (Exception e) {
            System.err.println("❌ Erreur : " + e.getMessage());
        }
    }
}
```

```java
// Avec POJOs (Plain Old Java Objects)
import org.bson.types.ObjectId;
import java.util.Date;

// Classe POJO
public class User {
    private ObjectId id;
    private String name;
    private String email;
    private Integer age;
    private Date createdAt;

    // Constructeurs
    public User() {}

    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createdAt = new Date();
    }

    // Getters et Setters
    public ObjectId getId() { return id; }
    public void setId(ObjectId id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }

    public Date getCreatedAt() { return createdAt; }
    public void setCreatedAt(Date createdAt) { this.createdAt = createdAt; }
}

// Utilisation avec POJOs
MongoCollection<User> userCollection = database
    .getCollection("users", User.class);

User newUser = new User("Alice", "alice@example.com");
newUser.setAge(28);

userCollection.insertOne(newUser);
System.out.println("User ID: " + newUser.getId());

// Recherche avec POJO
User foundUser = userCollection
    .find(Filters.eq("email", "alice@example.com"))
    .first();
```

```java
// Driver Reactive Streams
import com.mongodb.reactivestreams.client.MongoClients;
import com.mongodb.reactivestreams.client.MongoClient;
import com.mongodb.reactivestreams.client.MongoCollection;

import org.reactivestreams.Publisher;
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;

public class ReactiveMongoExample {

    public static void main(String[] args) {
        MongoClient mongoClient = MongoClients.create(
            "mongodb://localhost:27017"
        );

        MongoCollection<Document> collection = mongoClient
            .getDatabase("myapp")
            .getCollection("users");

        // Insert reactif
        Document user = new Document("name", "Bob")
            .append("email", "bob@example.com");

        Publisher<InsertOneResult> insertPublisher =
            collection.insertOne(user);

        insertPublisher.subscribe(new Subscriber<InsertOneResult>() {
            @Override
            public void onSubscribe(Subscription s) {
                s.request(1);
            }

            @Override
            public void onNext(InsertOneResult result) {
                System.out.println("Inséré: " + result.getInsertedId());
            }

            @Override
            public void onError(Throwable t) {
                System.err.println("Erreur: " + t.getMessage());
            }

            @Override
            public void onComplete() {
                System.out.println("Opération terminée");
                mongoClient.close();
            }
        });
    }
}
```

**Versions et compatibilité** :
- Driver 5.x : MongoDB 3.6+, Java 8+
- Driver 4.x : MongoDB 2.6+, Java 8+
- Spring Data MongoDB 4.x : Spring Boot 3.x

---

### 4. Driver C# / .NET

**Langage** : C#
**Package** : `MongoDB.Driver`
**Repository** : https://github.com/mongodb/mongo-csharp-driver
**Documentation** : https://mongodb.github.io/mongo-csharp-driver/

#### Caractéristiques

```bash
# NuGet Package Manager
Install-Package MongoDB.Driver

# .NET CLI
dotnet add package MongoDB.Driver

# Version minimale : .NET Standard 2.0, .NET Framework 4.7.2+
```

**Points forts** :
- ✅ Intégration parfaite avec l'écosystème .NET
- ✅ Support async/await natif
- ✅ LINQ to MongoDB (requêtes type-safe)
- ✅ Typage fort avec génériques
- ✅ Excellent pour applications d'entreprise Microsoft
- ✅ Support .NET Core et .NET Framework

**Points d'attention** :
- ⚠️ Écosystème principalement Microsoft
- ⚠️ Moins de ressources communautaires que Node.js/Python

#### Exemple d'initialisation

```csharp
using MongoDB.Driver;
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;
using System;
using System.Threading.Tasks;

// Classe POCO (Plain Old CLR Object)
public class User
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string Id { get; set; }

    [BsonElement("name")]
    public string Name { get; set; }

    [BsonElement("email")]
    public string Email { get; set; }

    [BsonElement("age")]
    [BsonIgnoreIfDefault]
    public int? Age { get; set; }

    [BsonElement("createdAt")]
    [BsonDateTimeOptions(Kind = DateTimeKind.Utc)]
    public DateTime CreatedAt { get; set; }
}

public class MongoDBService
{
    private readonly IMongoClient _client;
    private readonly IMongoDatabase _database;

    public MongoDBService(string connectionString, string databaseName)
    {
        var settings = MongoClientSettings.FromConnectionString(connectionString);

        // Configuration avancée
        settings.MaxConnectionPoolSize = 50;
        settings.MinConnectionPoolSize = 10;
        settings.MaxConnectionIdleTime = TimeSpan.FromSeconds(30);
        settings.ServerSelectionTimeout = TimeSpan.FromSeconds(5);
        settings.SocketTimeout = TimeSpan.FromSeconds(45);
        settings.RetryWrites = true;
        settings.RetryReads = true;

        _client = new MongoClient(settings);
        _database = _client.GetDatabase(databaseName);

        Console.WriteLine("✅ Connecté à MongoDB");
    }

    public IMongoCollection<User> Users =>
        _database.GetCollection<User>("users");

    // Méthode asynchrone
    public async Task<User> CreateUserAsync(User user)
    {
        user.CreatedAt = DateTime.UtcNow;
        await Users.InsertOneAsync(user);
        return user;
    }

    // Recherche avec LINQ
    public async Task<User> FindUserByEmailAsync(string email)
    {
        return await Users.Find(u => u.Email == email).FirstOrDefaultAsync();
    }
}

// Utilisation
class Program
{
    static async Task Main(string[] args)
    {
        var service = new MongoDBService(
            "mongodb://localhost:27017",
            "myapp"
        );

        try
        {
            // Créer un utilisateur
            var newUser = new User
            {
                Name = "Alice",
                Email = "alice@example.com",
                Age = 28
            };

            await service.CreateUserAsync(newUser);
            Console.WriteLine($"Utilisateur créé : {newUser.Id}");

            // Rechercher
            var foundUser = await service.FindUserByEmailAsync("alice@example.com");
            Console.WriteLine($"Utilisateur trouvé : {foundUser.Name}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Erreur : {ex.Message}");
        }
    }
}
```

```csharp
// LINQ to MongoDB - Requêtes complexes
using MongoDB.Driver.Linq;

public class UserRepository
{
    private readonly IMongoCollection<User> _users;

    public UserRepository(IMongoDatabase database)
    {
        _users = database.GetCollection<User>("users");
    }

    // Requête LINQ type-safe
    public async Task<List<User>> GetActiveAdultUsersAsync()
    {
        return await _users.AsQueryable()
            .Where(u => u.Age >= 18 && u.Status == "active")
            .OrderByDescending(u => u.CreatedAt)
            .Take(100)
            .ToListAsync();
    }

    // Aggregation avec LINQ
    public async Task<List<AgeGroup>> GetUsersByAgeGroupAsync()
    {
        return await _users.AsQueryable()
            .GroupBy(u => u.Age / 10 * 10) // Grouper par décennie
            .Select(g => new AgeGroup
            {
                AgeRange = g.Key,
                Count = g.Count(),
                AverageAge = g.Average(u => u.Age)
            })
            .ToListAsync();
    }
}
```

**Versions et compatibilité** :
- Driver 2.25+ : MongoDB 3.6+, .NET Standard 2.0+
- Driver 2.x : MongoDB 2.6+, .NET Framework 4.5.2+

---

### 5. Driver Go

**Langage** : Go
**Package** : `go.mongodb.org/mongo-driver`
**Repository** : https://github.com/mongodb/mongo-go-driver
**Documentation** : https://pkg.go.dev/go.mongodb.org/mongo-driver

#### Caractéristiques

```bash
# Installation
go get go.mongodb.org/mongo-driver/mongo

# Version minimale Go : 1.18+
```

**Points forts** :
- ✅ Performance exceptionnelle (concurrent natif)
- ✅ Faible empreinte mémoire
- ✅ Idéal pour microservices et APIs
- ✅ Compilation statique
- ✅ Excellente gestion de la concurrence (goroutines)
- ✅ Typage fort

**Points d'attention** :
- ⚠️ Gestion d'erreurs verbale (pas d'exceptions)
- ⚠️ Écosystème moins mature que Java/Node.js
- ⚠️ Marshaling/Unmarshaling manuel

#### Exemple d'initialisation

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.mongodb.org/mongo-driver/mongo/readpref"
)

// Struct pour User
type User struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Name      string             `bson:"name"`
    Email     string             `bson:"email"`
    Age       int                `bson:"age,omitempty"`
    CreatedAt time.Time          `bson:"createdAt"`
}

// Service MongoDB
type MongoService struct {
    client *mongo.Client
    db     *mongo.Database
}

func NewMongoService(uri string) (*MongoService, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Configuration du client
    clientOptions := options.Client().
        ApplyURI(uri).
        SetMaxPoolSize(50).
        SetMinPoolSize(10).
        SetMaxConnIdleTime(30 * time.Second).
        SetServerSelectionTimeout(5 * time.Second).
        SetSocketTimeout(45 * time.Second).
        SetRetryWrites(true).
        SetRetryReads(true)

    client, err := mongo.Connect(ctx, clientOptions)
    if err != nil {
        return nil, fmt.Errorf("erreur de connexion: %w", err)
    }

    // Vérifier la connexion
    if err = client.Ping(ctx, readpref.Primary()); err != nil {
        return nil, fmt.Errorf("échec du ping: %w", err)
    }

    fmt.Println("✅ Connecté à MongoDB")

    return &MongoService{
        client: client,
        db:     client.Database("myapp"),
    }, nil
}

func (s *MongoService) Close() error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    return s.client.Disconnect(ctx)
}

func (s *MongoService) Users() *mongo.Collection {
    return s.db.Collection("users")
}

// Créer un utilisateur
func (s *MongoService) CreateUser(ctx context.Context, user *User) error {
    user.CreatedAt = time.Now()

    result, err := s.Users().InsertOne(ctx, user)
    if err != nil {
        return fmt.Errorf("erreur d'insertion: %w", err)
    }

    user.ID = result.InsertedID.(primitive.ObjectID)
    return nil
}

// Rechercher par email
func (s *MongoService) FindUserByEmail(ctx context.Context, email string) (*User, error) {
    var user User

    filter := bson.M{"email": email}
    err := s.Users().FindOne(ctx, filter).Decode(&user)

    if err == mongo.ErrNoDocuments {
        return nil, nil // Pas trouvé
    }
    if err != nil {
        return nil, fmt.Errorf("erreur de recherche: %w", err)
    }

    return &user, nil
}

func main() {
    // Créer le service
    service, err := NewMongoService("mongodb://localhost:27017")
    if err != nil {
        log.Fatal(err)
    }
    defer service.Close()

    ctx := context.Background()

    // Créer un utilisateur
    newUser := &User{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   28,
    }

    if err := service.CreateUser(ctx, newUser); err != nil {
        log.Fatalf("Erreur création: %v", err)
    }

    fmt.Printf("Utilisateur créé : %s\n", newUser.ID.Hex())

    // Rechercher
    foundUser, err := service.FindUserByEmail(ctx, "alice@example.com")
    if err != nil {
        log.Fatalf("Erreur recherche: %v", err)
    }

    if foundUser != nil {
        fmt.Printf("Utilisateur trouvé : %s\n", foundUser.Name)
    }
}
```

```go
// Gestion avancée avec contextes
func (s *MongoService) UpdateUserWithTimeout(
    email string,
    updates bson.M,
) error {
    // Context avec timeout de 5 secondes
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    filter := bson.M{"email": email}
    update := bson.M{"$set": updates}

    result, err := s.Users().UpdateOne(ctx, filter, update)
    if err != nil {
        return fmt.Errorf("erreur de mise à jour: %w", err)
    }

    if result.MatchedCount == 0 {
        return fmt.Errorf("utilisateur non trouvé")
    }

    return nil
}
```

**Versions et compatibilité** :
- Driver 1.13+ : MongoDB 3.6+, Go 1.18+
- Driver 1.x : MongoDB 2.6+, Go 1.13+

---

### 6. Driver PHP

**Langage** : PHP
**Extension** : `mongodb` (extension C)
**Library** : `mongodb/mongodb` (abstraction PHP)
**Repository** : https://github.com/mongodb/mongo-php-library
**Documentation** : https://www.php.net/manual/en/set.mongodb.php

#### Caractéristiques

```bash
# Installation de l'extension (requis)
pecl install mongodb

# Composer (library PHP)
composer require mongodb/mongodb

# Version minimale PHP : 7.4+
```

**Points forts** :
- ✅ Extension C pour performance
- ✅ Intégration facile avec applications web PHP
- ✅ Compatible Symfony, Laravel
- ✅ API simple et intuitive

**Points d'attention** :
- ⚠️ Nécessite deux composants (extension + library)
- ⚠️ Pas de support async natif

#### Exemple d'initialisation

```php
<?php
require 'vendor/autoload.php';

use MongoDB\Client;
use MongoDB\BSON\UTCDateTime;
use MongoDB\Exception\Exception;

// Configuration du client
$client = new Client(
    'mongodb://localhost:27017',
    [
        'maxPoolSize' => 50,
        'minPoolSize' => 10,
        'serverSelectionTimeoutMS' => 5000,
        'socketTimeoutMS' => 45000,
    ]
);

try {
    // Vérifier la connexion
    $client->selectDatabase('admin')->command(['ping' => 1]);
    echo "✅ Connecté à MongoDB\n";

    // Accès à la base et collection
    $database = $client->selectDatabase('myapp');
    $users = $database->selectCollection('users');

    // Insertion
    $result = $users->insertOne([
        'name' => 'Alice',
        'email' => 'alice@example.com',
        'age' => 28,
        'createdAt' => new UTCDateTime()
    ]);

    echo "Document inséré : " . $result->getInsertedId() . "\n";

    // Recherche
    $user = $users->findOne(['email' => 'alice@example.com']);
    echo "Utilisateur trouvé : " . $user['name'] . "\n";

} catch (Exception $e) {
    echo "❌ Erreur : " . $e->getMessage() . "\n";
}
```

---

### 7. Driver Ruby

**Langage** : Ruby
**Gem** : `mongo`
**Repository** : https://github.com/mongodb/mongo-ruby-driver
**Documentation** : https://mongodb.com/docs/ruby-driver/

#### Caractéristiques

```ruby
# Gemfile
gem 'mongo', '~> 2.19'

# Installation
bundle install

# Version minimale Ruby : 2.5+
```

**Points forts** :
- ✅ Syntaxe Ruby idiomatique
- ✅ Intégration Rails via Mongoid
- ✅ API élégante et expressive

#### Exemple d'initialisation

```ruby
require 'mongo'

# Configuration du client
client = Mongo::Client.new(
  ['localhost:27017'],
  database: 'myapp',
  max_pool_size: 50,
  min_pool_size: 10,
  server_selection_timeout: 5,
  socket_timeout: 45
)

begin
  # Vérifier la connexion
  client.database.command(ping: 1)
  puts '✅ Connecté à MongoDB'

  # Accès à la collection
  users = client[:users]

  # Insertion
  result = users.insert_one(
    name: 'Alice',
    email: 'alice@example.com',
    age: 28,
    created_at: Time.now
  )

  puts "Document inséré : #{result.inserted_id}"

  # Recherche
  user = users.find(email: 'alice@example.com').first
  puts "Utilisateur trouvé : #{user['name']}"

rescue Mongo::Error => e
  puts "❌ Erreur : #{e.message}"
ensure
  client.close
end
```

---

## Tableau comparatif détaillé

| Critère | Node.js | Python | Java | C# | Go | PHP | Ruby |
|---------|---------|--------|------|----|----|-----|------|
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Async natif** | ✅ | ⚠️ Motor | ✅ | ✅ | ✅ | ❌ | ⚠️ |
| **Type safety** | ⚠️ TS | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Facilité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Écosystème** | Très riche | Très riche | Riche | Riche | Croissant | Riche | Moyen |
| **Maturité** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **ODM/ORM** | Mongoose | MongoEngine | Spring Data | - | - | - | Mongoid |
| **Cas d'usage** | Web, API, Real-time | Data Science, ML | Entreprise | Entreprise MS | Microservices | Web PHP | Web Rails |

## Comment choisir votre driver ?

### Critères de décision

1. **Stack technologique existante**
   - Choisissez le driver de votre stack principale
   - Considérez les compétences de votre équipe

2. **Type d'application**
   - **Web/API REST** : Node.js, Go, Python
   - **Microservices** : Go, Java, Node.js
   - **Entreprise** : Java, C#
   - **Data Science** : Python
   - **Real-time** : Node.js

3. **Performance requise**
   - **Haute performance** : Go, Java
   - **Concurrence élevée** : Go, Java, Node.js
   - **Low latency** : Go, Java

4. **Complexité fonctionnelle**
   - **Simple** : Node.js, Python, PHP
   - **Complexe** : Java (Spring), C#

## Matrice de compatibilité MongoDB

| Driver Version | MongoDB 3.6 | MongoDB 4.0 | MongoDB 4.2 | MongoDB 4.4 | MongoDB 5.0 | MongoDB 6.0 | MongoDB 7.0 |
|----------------|-------------|-------------|-------------|-------------|-------------|-------------|-------------|
| Node.js 6.x | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| PyMongo 4.x | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Java 5.x | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| C# 2.25+ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Go 1.13+ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

## Bonnes pratiques communes

### ✅ 1. Toujours utiliser la dernière version stable

```bash
# Vérifier les versions
npm outdated mongodb
pip list --outdated | grep pymongo
mvn versions:display-dependency-updates
```

### ✅ 2. Lire la documentation de migration

Avant de mettre à jour un driver majeur, consultez :
- Les breaking changes
- Les nouvelles fonctionnalités
- Les dépréciations

### ✅ 3. Tester en environnement de staging

```javascript
// Exemple de tests d'intégration
describe('MongoDB Connection', () => {
    it('should connect successfully', async () => {
        const client = new MongoClient(uri);
        await client.connect();
        const result = await client.db().admin().ping();
        expect(result.ok).toBe(1);
        await client.close();
    });
});
```

### ✅ 4. Monitorer les performances du driver

- Utiliser les outils de monitoring natifs
- Surveiller les métriques de connection pool
- Tracer les requêtes lentes

## Ressources officielles

| Langage | Documentation | API Reference | GitHub |
|---------|--------------|---------------|--------|
| Node.js | [docs](https://mongodb.github.io/node-mongodb-native/) | [API](https://mongodb.github.io/node-mongodb-native/5.9/classes/MongoClient.html) | [repo](https://github.com/mongodb/node-mongodb-native) |
| Python | [docs](https://pymongo.readthedocs.io/) | [API](https://pymongo.readthedocs.io/en/stable/api/) | [repo](https://github.com/mongodb/mongo-python-driver) |
| Java | [docs](https://mongodb.github.io/mongo-java-driver/) | [API](https://mongodb.github.io/mongo-java-driver/5.1/apidocs/) | [repo](https://github.com/mongodb/mongo-java-driver) |
| C# | [docs](https://mongodb.github.io/mongo-csharp-driver/) | [API](https://mongodb.github.io/mongo-csharp-driver/2.25/apidocs/) | [repo](https://github.com/mongodb/mongo-csharp-driver) |
| Go | [docs](https://www.mongodb.com/docs/drivers/go/current/) | [pkg](https://pkg.go.dev/go.mongodb.org/mongo-driver) | [repo](https://github.com/mongodb/mongo-go-driver) |

## Conclusion

Le choix du driver MongoDB est une décision stratégique qui impacte :
- Les performances de votre application
- La productivité de votre équipe
- La maintenabilité du code
- La scalabilité du système

Tous les drivers officiels sont :
- ✅ Maintenus activement par MongoDB Inc.
- ✅ Compatibles avec les dernières versions de MongoDB
- ✅ Performants et optimisés
- ✅ Documentés et supportés

Le "meilleur" driver est celui qui s'intègre naturellement dans votre stack technologique existante et répond à vos besoins spécifiques de performance et de fonctionnalités.

---

**Section suivante** : 15.2 Driver Node.js / JavaScript

Dans la prochaine section, nous plongerons en profondeur dans le driver Node.js, explorant ses fonctionnalités avancées, ses patterns d'utilisation, et les meilleures pratiques pour des applications Node.js performantes et robustes.

⏭️ [Driver Node.js / JavaScript](/15-drivers-integration-applicative/02-driver-nodejs-javascript.md)
