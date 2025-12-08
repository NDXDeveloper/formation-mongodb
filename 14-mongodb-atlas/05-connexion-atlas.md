🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.5 Connexion à Atlas

## Introduction

La connexion à MongoDB Atlas est le **point de départ** de toute interaction avec votre base de données. Cette section couvre les connection strings, les drivers officiels, les stratégies de connection pooling, la gestion des credentials, et le troubleshooting. Pour les équipes DevOps, maîtriser ces aspects est essentiel pour construire des applications fiables, performantes et sécurisées.

### 🎯 Objectifs de cette Section

- Comprendre les formats de connection strings (Standard, SRV, options)
- Configurer correctement les drivers MongoDB
- Implémenter le connection pooling pour la performance
- Gérer les credentials de manière sécurisée
- Diagnostiquer et résoudre les problèmes de connexion
- Appliquer les best practices par environnement

---

## 🔗 Connection Strings

### Anatomie d'une Connection String

```
┌───────────────────────────────────────────────────────────────────────┐
│                   MONGODB CONNECTION STRING ANATOMY                   │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  FORMAT STANDARD:                                                     │
│  mongodb://[username:password@]host[:port][/database][?options]       │
│                                                                       │
│  FORMAT SRV (Recommended):                                            │
│  mongodb+srv://[username:password@]host[/database][?options]          │
│                                                                       │
│  ─────────────────────────────────────────────────────────────────────│
│                                                                       │
│  EXEMPLE ATLAS (SRV):                                                 │
│  mongodb+srv://myuser:mypassword@cluster0.xxxxx.mongodb.net/          │
│              mydb?retryWrites=true&w=majority                         │
│                                                                       │
│  Décomposition:                                                       │
│  ┌────────────┐                                                       │
│  │ mongodb+srv│ → Protocole SRV (DNS lookup)                          │
│  └────────────┘                                                       │
│       ┌──────────────────────┐                                        │
│       │ myuser:mypassword    │ → Credentials                          │
│       └──────────────────────┘                                        │
│                 ┌────────────────────────────────┐                    │
│                 │ cluster0.xxxxx.mongodb.net     │ → Hostname         │
│                 └────────────────────────────────┘                    │
│                                           ┌────┐                      │
│                                           │mydb│ → Default database   │
│                                           └────┘                      │
│                                                 ┌──────────────────┐  │
│                                                 │ ?retryWrites=... │  │
│                                                 │ Options          │  │
│                                                 └──────────────────┘  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Standard vs SRV Format

```
┌────────────────────────────────────────────────────────────────────────┐
│              STANDARD vs SRV CONNECTION STRING                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  STANDARD FORMAT (Explicit hosts):                                     │
│  mongodb://myuser:pass@cluster0-shard-00-00.xxxxx.mongodb.net:27017,   │
│            cluster0-shard-00-01.xxxxx.mongodb.net:27017,               │
│            cluster0-shard-00-02.xxxxx.mongodb.net:27017/               │
│            mydb?replicaSet=atlas-abc123-shard-0&ssl=true               │
│                                                                        │
│  ✅ Avantages:                                                         │
│  • Contrôle explicite des hosts                                        │
│  • Pas de dépendance DNS                                               │
│                                                                        │
│  ❌ Inconvénients:                                                     │
│  • Long et verbeux                                                     │
│  • Doit être mis à jour si topology change                             │
│  • Difficile à maintenir                                               │
│                                                                        │
│  ───────────────────────────────────────────────────────────────────── │
│                                                                        │
│  SRV FORMAT (DNS-based, Recommended):                                  │
│  mongodb+srv://myuser:pass@cluster0.xxxxx.mongodb.net/mydb             │
│                                                                        │
│  ✅ Avantages:                                                         │
│  • Court et lisible                                                    │
│  • DNS SRV lookup automatique → découverte des hosts                   │
│  • Résistant aux changements de topology                               │
│  • TLS/SSL activé par défaut                                           │
│  • Options par défaut optimales                                        │
│                                                                        │
│  ❌ Inconvénients:                                                     │
│  • Nécessite résolution DNS                                            │
│  • Peut échouer si DNS non disponible                                  │
│                                                                        │
│  RECOMMANDATION: ✅ Toujours utiliser SRV pour Atlas                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Connection String selon Configuration Réseau

```yaml
# 1. PUBLIC INTERNET (avec IP Whitelist)
mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/mydb?retryWrites=true&w=majority

# 2. VPC PEERING (Internal IPs)
# Connection string identique, mais résolution DNS pointe vers IPs privées
mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/mydb?retryWrites=true&w=majority

# 3. PRIVATELINK/PRIVATE ENDPOINT
# Hostname spécifique pour private endpoint
mongodb+srv://username:password@cluster0-pl-0.xxxxx.mongodb.net/mydb?retryWrites=true&w=majority

# Note: Le suffixe "-pl-" indique un Private Link endpoint
```

### Options de Connection String Importantes

```
┌────────────────────────────────────────────────────────────────────────┐
│              IMPORTANT CONNECTION STRING OPTIONS                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  OPTION                   DEFAULT    DESCRIPTION                       │
│  ───────────────────────────────────────────────────────────────────── │
│  retryWrites              true       Retry transient write errors      │
│  w                        majority   Write concern (durability)        │
│  readPreference           primary    Read routing strategy             │
│  maxPoolSize              100        Max connections in pool           │
│  minPoolSize              0          Min connections to maintain       │
│  maxIdleTimeMS            0          Max idle time before close        │
│  connectTimeoutMS         10000      Connection timeout (10s)          │
│  socketTimeoutMS          0          Socket read timeout               │
│  serverSelectionTimeoutMS 30000      Server selection timeout (30s)    │
│  heartbeatFrequencyMS     10000      Monitoring frequency (10s)        │
│  appName                  null       Application identifier            │
│  compressors              snappy     Compression algorithms            │
│  zlibCompressionLevel     6          Zlib compression level            │
│  tls/ssl                  true       TLS encryption (auto with SRV)    │
│  authSource               admin      Authentication database           │
│  authMechanism            SCRAM      Authentication mechanism          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Exemples de Connection Strings Optimisées

```javascript
// ✅ PRODUCTION - Highly Available, Durable
const uri = "mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/mydb?" +
  "retryWrites=true&" +
  "w=majority&" +
  "readPreference=primaryPreferred&" +  // Failover to secondary
  "maxPoolSize=50&" +                   // Connection pool
  "minPoolSize=10&" +
  "maxIdleTimeMS=60000&" +              // Close idle after 60s
  "connectTimeoutMS=10000&" +
  "serverSelectionTimeoutMS=5000&" +
  "appName=MyApp-Production";

// ✅ READ-HEAVY - Analytics, Reporting
const uri = "mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/mydb?" +
  "readPreference=secondaryPreferred&" + // Prefer secondaries
  "maxStalenessSeconds=120&" +           // Allow 2min stale reads
  "maxPoolSize=100&" +
  "compressors=snappy,zlib&" +           // Compression for large reads
  "appName=MyApp-Analytics";

// ✅ LOW-LATENCY - Time-critical operations
const uri = "mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/mydb?" +
  "retryWrites=true&" +
  "w=1&" +                               // Acknowledge from primary only
  "readPreference=primary&" +
  "maxPoolSize=200&" +
  "connectTimeoutMS=5000&" +
  "serverSelectionTimeoutMS=3000&" +
  "appName=MyApp-Realtime";

// ⚠️ DEVELOPMENT - Relaxed for debugging
const uri = "mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/mydb?" +
  "retryWrites=false&" +                 // See errors immediately
  "w=1&" +
  "readPreference=primary&" +
  "maxPoolSize=10&" +
  "appName=MyApp-Dev";
```

---

## 🔌 Drivers et Compatibilité

### Drivers Officiels MongoDB

```
┌────────────────────────────────────────────────────────────────────────┐
│                    OFFICIAL MONGODB DRIVERS                            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  LANGUAGE       DRIVER              LATEST VERSION    ATLAS SUPPORT    │
│  ───────────────────────────────────────────────────────────────────── │
│  Node.js        mongodb             6.x               ✅ Full          │
│  Python         pymongo             4.x               ✅ Full          │
│  Java           mongodb-driver      5.x               ✅ Full          │
│  C#/.NET        MongoDB.Driver      2.x               ✅ Full          │
│  Go             mongo-go-driver     1.x               ✅ Full          │
│  PHP            mongodb             1.x               ✅ Full          │
│  Ruby           mongo               2.x               ✅ Full          │
│  Rust           mongodb             2.x               ✅ Full          │
│  C              mongo-c-driver      1.x               ✅ Full          │
│  C++            mongo-cxx-driver    3.x               ✅ Full          │
│  Scala          mongo-scala-driver  5.x               ✅ Full          │
│                                                                        │
│  ALL DRIVERS:                                                          │
│  • Support SRV connection strings                                      │
│  • Automatic server discovery                                          │
│  • Connection pooling                                                  │
│  • Automatic failover                                                  │
│  • TLS/SSL support                                                     │
│  • Compression                                                         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Exemples de Connexion par Langage

#### Node.js (MongoDB Driver)

```javascript
// Installation: npm install mongodb
const { MongoClient } = require('mongodb');

// Configuration
const uri = process.env.MONGODB_URI;
const options = {
  maxPoolSize: 50,
  minPoolSize: 10,
  socketTimeoutMS: 45000,
  family: 4, // Use IPv4, skip trying IPv6
  retryWrites: true,
  retryReads: true,
  w: 'majority',
};

// Create client
const client = new MongoClient(uri, options);

// Connect
async function run() {
  try {
    await client.connect();
    console.log('Connected to Atlas');

    const database = client.db('mydb');
    const collection = database.collection('users');

    // Operations
    const user = await collection.findOne({ email: 'user@example.com' });
    console.log(user);

  } catch (error) {
    console.error('Connection error:', error);
  } finally {
    // Don't close in production (keep connection pool open)
    // await client.close();
  }
}

run();

// Graceful shutdown
process.on('SIGINT', async () => {
  await client.close();
  console.log('MongoDB connection closed');
  process.exit(0);
});
```

#### Python (PyMongo)

```python
# Installation: pip install pymongo[srv]
from pymongo import MongoClient
from pymongo.errors import ConnectionFailure, ServerSelectionTimeoutError
import os

# Configuration
uri = os.environ.get('MONGODB_URI')
options = {
    'maxPoolSize': 50,
    'minPoolSize': 10,
    'socketTimeoutMS': 45000,
    'retryWrites': True,
    'retryReads': True,
    'w': 'majority',
    'appName': 'MyPythonApp'
}

# Create client
client = MongoClient(uri, **options)

# Test connection
try:
    # Ping to verify connection
    client.admin.command('ping')
    print('Connected to Atlas')

    # Get database and collection
    db = client['mydb']
    collection = db['users']

    # Operations
    user = collection.find_one({'email': 'user@example.com'})
    print(user)

except ConnectionFailure as e:
    print(f'Connection failed: {e}')
except ServerSelectionTimeoutError as e:
    print(f'Server selection timeout: {e}')
finally:
    # Close connection when done
    client.close()
```

#### Java (MongoDB Driver)

```java
// Maven dependency:
// <dependency>
//   <groupId>org.mongodb</groupId>
//   <artifactId>mongodb-driver-sync</artifactId>
//   <version>5.0.0</version>
// </dependency>

import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

import java.util.concurrent.TimeUnit;

public class AtlasConnection {
    public static void main(String[] args) {
        // Configuration
        String uri = System.getenv("MONGODB_URI");

        ConnectionString connectionString = new ConnectionString(uri);

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .applyToConnectionPoolSettings(builder ->
                builder.maxSize(50)
                       .minSize(10)
                       .maxConnectionIdleTime(60, TimeUnit.SECONDS))
            .applyToSocketSettings(builder ->
                builder.connectTimeout(10, TimeUnit.SECONDS)
                       .readTimeout(45, TimeUnit.SECONDS))
            .retryWrites(true)
            .retryReads(true)
            .applicationName("MyJavaApp")
            .build();

        // Create client
        try (MongoClient mongoClient = MongoClients.create(settings)) {
            // Test connection
            mongoClient.getDatabase("admin").runCommand(new Document("ping", 1));
            System.out.println("Connected to Atlas");

            // Get database and collection
            MongoDatabase database = mongoClient.getDatabase("mydb");
            MongoCollection<Document> collection = database.getCollection("users");

            // Operations
            Document user = collection.find(new Document("email", "user@example.com"))
                .first();
            System.out.println(user);

        } catch (Exception e) {
            System.err.println("Connection error: " + e.getMessage());
        }
    }
}
```

#### Go (mongo-go-driver)

```go
// Installation: go get go.mongodb.org/mongo-driver/mongo
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
    // Configuration
    uri := os.Getenv("MONGODB_URI")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Client options
    clientOptions := options.Client().ApplyURI(uri).
        SetMaxPoolSize(50).
        SetMinPoolSize(10).
        SetMaxConnIdleTime(60 * time.Second).
        SetRetryWrites(true).
        SetRetryReads(true).
        SetAppName("MyGoApp")

    // Connect to Atlas
    client, err := mongo.Connect(ctx, clientOptions)
    if err != nil {
        log.Fatal(err)
    }
    defer client.Disconnect(ctx)

    // Ping to verify connection
    err = client.Ping(ctx, nil)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Connected to Atlas")

    // Get database and collection
    collection := client.Database("mydb").Collection("users")

    // Operations
    var result bson.M
    filter := bson.D{{"email", "user@example.com"}}
    err = collection.FindOne(ctx, filter).Decode(&result)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(result)
}
```

#### C# / .NET (MongoDB.Driver)

```csharp
// NuGet: Install-Package MongoDB.Driver
using MongoDB.Driver;
using MongoDB.Bson;
using System;

class Program
{
    static async Task Main(string[] args)
    {
        // Configuration
        var uri = Environment.GetEnvironmentVariable("MONGODB_URI");

        var settings = MongoClientSettings.FromConnectionString(uri);
        settings.MaxConnectionPoolSize = 50;
        settings.MinConnectionPoolSize = 10;
        settings.MaxConnectionIdleTime = TimeSpan.FromSeconds(60);
        settings.RetryWrites = true;
        settings.RetryReads = true;
        settings.ApplicationName = "MyCSharpApp";

        // Create client
        var client = new MongoClient(settings);

        try
        {
            // Ping to verify connection
            await client.GetDatabase("admin").RunCommandAsync(
                (Command<BsonDocument>)"{ping:1}"
            );
            Console.WriteLine("Connected to Atlas");

            // Get database and collection
            var database = client.GetDatabase("mydb");
            var collection = database.GetCollection<BsonDocument>("users");

            // Operations
            var filter = Builders<BsonDocument>.Filter.Eq("email", "user@example.com");
            var user = await collection.Find(filter).FirstOrDefaultAsync();
            Console.WriteLine(user);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Connection error: {ex.Message}");
        }
    }
}
```

---

## 🏊 Connection Pooling

### Architecture Connection Pool

```
┌───────────────────────────────────────────────────────────────────────┐
│                    CONNECTION POOLING ARCHITECTURE                    │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   APPLICATION                     CONNECTION POOL                     │
│   ┌────────────────┐              ┌────────────────────────────────┐  │
│   │                │              │                                │  │
│   │  Request 1 ────┼──────────────┼─► [Conn 1] ──────┐             │  │
│   │                │              │                  │             │  │
│   │  Request 2 ────┼──────────────┼─► [Conn 2] ──────┤             │  │
│   │                │              │                  │             │  │
│   │  Request 3 ────┼──────────────┼─► [Conn 3] ──────┤             │  │
│   │                │              │                   ▼            │  │
│   │  Request 4 ────┼─────X────────┼   [Conn 4]   MongoDB Atlas     │  │
│   │  (waits)       │   All busy   │                   ▲            │  │
│   │                │              │   [Conn 5] ──────┤             │  │
│   │                │              │                  │             │  │
│   │                │              │   [Conn 6] ──────┤             │  │
│   │                │              │                  │             │  │
│   │                │              │   [Idle]    ─────┘             │  │
│   │                │              │                                │  │
│   └────────────────┘              │   Min Pool Size: 10            │  │
│                                   │   Max Pool Size: 50            │  │
│                                   │   Current: 6 active, 4 idle    │  │
│                                   └────────────────────────────────┘  │
│                                                                       │
│   LIFECYCLE:                                                          │
│   1. Request arrives → Get connection from pool                       │
│   2. If available → Reuse existing connection                         │
│   3. If none available and < maxPoolSize → Create new connection      │
│   4. If at maxPoolSize → Wait in queue                                │
│   5. After operation → Return connection to pool (don't close)        │
│   6. Idle connections (> maxIdleTime) → Closed automatically          │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Configuration Optimale par Use Case

```javascript
// 1. WEB APPLICATION (High Traffic)
const webAppConfig = {
  maxPoolSize: 100,           // Handle bursts
  minPoolSize: 20,            // Keep connections warm
  maxIdleTimeMS: 30000,       // Close idle after 30s
  waitQueueTimeoutMS: 5000,   // Fail fast if pool exhausted
};

// 2. MICROSERVICE (Moderate Traffic)
const microserviceConfig = {
  maxPoolSize: 50,
  minPoolSize: 10,
  maxIdleTimeMS: 60000,       // 1 minute
  waitQueueTimeoutMS: 3000,
};

// 3. BATCH PROCESSING (Long-running)
const batchConfig = {
  maxPoolSize: 10,            // Low concurrency
  minPoolSize: 5,
  maxIdleTimeMS: 300000,      // 5 minutes (long operations)
  socketTimeoutMS: 300000,    // Don't timeout during processing
};

// 4. SERVERLESS FUNCTION (AWS Lambda, etc.)
const serverlessConfig = {
  maxPoolSize: 1,             // Lambda = 1 concurrent execution
  minPoolSize: 0,             // Don't keep connections
  maxIdleTimeMS: 10000,       // Close quickly
  serverSelectionTimeoutMS: 5000,  // Fail fast
};

// 5. ANALYTICS / REPORTING (Read-heavy)
const analyticsConfig = {
  maxPoolSize: 200,           // Many concurrent reads
  minPoolSize: 50,
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 120,
};
```

### Monitoring Connection Pool

```javascript
// Node.js: Monitor connection pool events
const client = new MongoClient(uri, options);

client.on('connectionPoolCreated', (event) => {
  console.log('Pool created:', event.options);
});

client.on('connectionCreated', (event) => {
  console.log('Connection created:', event.connectionId);
});

client.on('connectionReady', (event) => {
  console.log('Connection ready:', event.connectionId);
});

client.on('connectionCheckedOut', (event) => {
  console.log('Connection checked out:', event.connectionId);
});

client.on('connectionCheckedIn', (event) => {
  console.log('Connection checked in:', event.connectionId);
});

client.on('connectionClosed', (event) => {
  console.log('Connection closed:', event.connectionId, event.reason);
});

// Métriques utiles
function logPoolStats(client) {
  setInterval(() => {
    const stats = client.topology?.s?.servers?.values();
    if (stats) {
      for (const server of stats) {
        console.log({
          host: server.description.address,
          availableConnections: server.pool.availableConnectionCount,
          totalConnections: server.pool.totalConnectionCount,
          waitQueueSize: server.pool.waitQueueSize,
        });
      }
    }
  }, 30000); // Every 30 seconds
}
```

---

## 🔐 Gestion Sécurisée des Credentials

### Mauvaises Pratiques (à ÉVITER)

```javascript
// ❌ NEVER: Hardcoded credentials
const uri = "mongodb+srv://admin:Password123@cluster0.xxxxx.mongodb.net/";

// ❌ NEVER: Committed to Git
// .env file:
MONGODB_URI=mongodb+srv://admin:Password123@cluster0.xxxxx.mongodb.net/

// ❌ NEVER: Logged in plain text
console.log(`Connecting to: ${uri}`);  // Logs password!
```

### Bonnes Pratiques

```
┌───────────────────────────────────────────────────────────────────────┐
│                SECURE CREDENTIAL MANAGEMENT STRATEGIES                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. ENVIRONMENT VARIABLES (Minimum)                                   │
│     ┌────────────────────────────────────────────────────────────────┐│
│     │ # .env (never commit)                                          ││
│     │ MONGODB_URI=mongodb+srv://user:pass@cluster0.xxx.mongodb.net/  ││
│     │                                                                ││
│     │ # .gitignore                                                   ││
│     │ .env                                                           ││
│     │ .env.*                                                         ││
│     │                                                                ││
│     │ # app.js                                                       ││
│     │ require('dotenv').config();                                    ││
│     │ const uri = process.env.MONGODB_URI;                           ││
│     └────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  2. SECRETS MANAGER (AWS, Azure, GCP)                                 │
│     ┌────────────────────────────────────────────────────────────────┐│
│     │ // AWS Secrets Manager                                         ││
│     │ const AWS = require('aws-sdk');                                ││
│     │ const secretsManager = new AWS.SecretsManager();               ││
│     │                                                                ││
│     │ async function getSecret() {                                   ││
│     │   const data = await secretsManager.getSecretValue({           ││
│     │     SecretId: 'prod/mongodb/uri'                               ││
│     │   }).promise();                                                ││
│     │   return JSON.parse(data.SecretString).uri;                    ││
│     │ }                                                              ││
│     └────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  3. VAULT (HashiCorp Vault)                                           │
│     ┌────────────────────────────────────────────────────────────────┐│
│     │ const vault = require('node-vault')();                         ││
│     │                                                                ││
│     │ async function getMongoUri() {                                 ││
│     │   const result = await vault.read(                             ││
│     │     'secret/data/mongodb/production'                           ││
│     │   );                                                           ││
│     │   return result.data.data.uri;                                 ││
│     │ }                                                              ││
│     └────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  4. KUBERNETES SECRETS                                                │
│     ┌────────────────────────────────────────────────────────────────┐│
│     │ # k8s secret                                                   ││
│     │ apiVersion: v1                                                 ││
│     │ kind: Secret                                                   ││
│     │ metadata:                                                      ││
│     │   name: mongodb-secret                                         ││
│     │ type: Opaque                                                   ││
│     │ stringData:                                                    ││
│     │   uri: mongodb+srv://...                                       ││
│     │                                                                ││
│     │ # Deployment                                                   ││
│     │ env:                                                           ││
│     │   - name: MONGODB_URI                                          ││
│     │     valueFrom:                                                 ││
│     │       secretKeyRef:                                            ││
│     │         name: mongodb-secret                                   ││
│     │         key: uri                                               ││
│     └────────────────────────────────────────────────────────────────┘│
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Exemple Complet : AWS Secrets Manager

```javascript
// secrets.js - Wrapper for AWS Secrets Manager
const AWS = require('aws-sdk');

class SecretsManager {
  constructor() {
    this.client = new AWS.SecretsManager({
      region: process.env.AWS_REGION || 'us-east-1'
    });
    this.cache = new Map();
    this.cacheTTL = 300000; // 5 minutes
  }

  async getSecret(secretName) {
    // Check cache
    const cached = this.cache.get(secretName);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.value;
    }

    // Fetch from AWS
    try {
      const data = await this.client.getSecretValue({
        SecretId: secretName
      }).promise();

      const secret = data.SecretString ?
        JSON.parse(data.SecretString) :
        Buffer.from(data.SecretBinary, 'base64').toString('ascii');

      // Cache it
      this.cache.set(secretName, {
        value: secret,
        timestamp: Date.now()
      });

      return secret;
    } catch (error) {
      console.error('Error fetching secret:', error);
      throw error;
    }
  }

  async getMongoUri(environment) {
    const secretName = `${environment}/mongodb/atlas`;
    const secret = await this.getSecret(secretName);
    return secret.uri;
  }
}

module.exports = new SecretsManager();

// app.js - Usage
const { MongoClient } = require('mongodb');
const secrets = require('./secrets');

async function connectToMongo() {
  const uri = await secrets.getMongoUri('production');
  const client = new MongoClient(uri);
  await client.connect();
  return client;
}
```

---

## 🔍 Troubleshooting Connexion

### Diagnostic de Problèmes Courants

```
┌────────────────────────────────────────────────────────────────────────┐
│                  CONNECTION TROUBLESHOOTING GUIDE                      │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ERROR                            CAUSE              SOLUTION          │
│  ───────────────────────────────────────────────────────────────────── │
│  "Connection timeout"             • IP not whitelisted                 │
│  MongoServerSelectionError        • Firewall blocking                  │
│                                   • Wrong hostname                     │
│                                   → Add IP to Access List              │
│                                   → Check security groups              │
│                                   → Verify connection string           │
│                                                                        │
│  "Authentication failed"          • Wrong credentials                  │
│  MongoServerError: auth failed    • User doesn't exist                 │
│                                   • Wrong auth database                │
│                                   → Verify username/password           │
│                                   → Check authSource in URI            │
│                                   → Create user if missing             │
│                                                                        │
│  "SSL handshake failed"           • TLS version mismatch               │
│  MongoServerError: SSL            • Certificate issues                 │
│                                   → Update driver to latest            │
│                                   → Check tls=true in URI              │
│                                   → Verify system has CA certs         │
│                                                                        │
│  "No primary found"               • Replica set issue                  │
│  MongoServerError                 • All nodes down                     │
│                                   • Network partition                  │
│                                   → Check Atlas status page            │
│                                   → Wait for failover (~30s)           │
│                                   → Verify replica set name            │
│                                                                        │
│  "Too many connections"           • Pool exhausted                     │
│  MongoServerError: too many       • Connection leak                    │
│                                   • Pool size too small                │
│                                   → Increase maxPoolSize               │
│                                   → Fix connection leaks               │
│                                   → Implement connection reuse         │
│                                                                        │
│  "DNS resolution failed"          • SRV lookup failed                  │
│  ENOTFOUND                        • DNS server issue                   │
│                                   • Incorrect hostname                 │
│                                   → Check DNS resolution:              │
│                                     nslookup cluster0.xxx.mongodb.net  │
│                                   → Try standard format (not SRV)      │
│                                   → Check /etc/resolv.conf             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Outils de Diagnostic

```bash
# 1. Test DNS resolution
nslookup cluster0.xxxxx.mongodb.net
dig +short SRV _mongodb._tcp.cluster0.xxxxx.mongodb.net

# 2. Test network connectivity
nc -zv cluster0-shard-00-00.xxxxx.mongodb.net 27017
telnet cluster0-shard-00-00.xxxxx.mongodb.net 27017

# 3. Test TLS/SSL
openssl s_client -connect cluster0-shard-00-00.xxxxx.mongodb.net:27017 \
  -starttls mongodb

# 4. Test authentication with mongosh
mongosh "mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/test"

# 5. Check if IP is whitelisted
curl https://ifconfig.me  # Get your public IP
# Then check Atlas IP Access List

# 6. Verbose connection logging (Node.js)
DEBUG=* node app.js
# or
MONGODB_LOG_LEVEL=debug node app.js
```

### Script de Test de Connexion

```javascript
// connection-test.js - Comprehensive connection test
const { MongoClient } = require('mongodb');

async function testConnection(uri) {
  console.log('Testing MongoDB Atlas connection...\n');

  const client = new MongoClient(uri, {
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 5000,
  });

  try {
    // 1. Test connection
    console.log('1. Connecting...');
    await client.connect();
    console.log('✅ Connection successful\n');

    // 2. Test ping
    console.log('2. Testing ping...');
    const pingResult = await client.db('admin').command({ ping: 1 });
    console.log('✅ Ping successful:', pingResult, '\n');

    // 3. Get server info
    console.log('3. Getting server info...');
    const serverInfo = await client.db('admin').command({ buildInfo: 1 });
    console.log('✅ MongoDB version:', serverInfo.version);
    console.log('✅ Server:', serverInfo.modules, '\n');

    // 4. Test database access
    console.log('4. Testing database access...');
    const databases = await client.db().admin().listDatabases();
    console.log('✅ Accessible databases:', databases.databases.map(d => d.name), '\n');

    // 5. Test write operation
    console.log('5. Testing write operation...');
    const testDb = client.db('test');
    const testCol = testDb.collection('connection_test');
    const writeResult = await testCol.insertOne({
      test: true,
      timestamp: new Date()
    });
    console.log('✅ Write successful, ID:', writeResult.insertedId, '\n');

    // 6. Test read operation
    console.log('6. Testing read operation...');
    const doc = await testCol.findOne({ _id: writeResult.insertedId });
    console.log('✅ Read successful:', doc, '\n');

    // 7. Cleanup
    await testCol.deleteOne({ _id: writeResult.insertedId });

    console.log('✅ All tests passed! Connection is healthy.\n');

  } catch (error) {
    console.error('❌ Connection test failed:');
    console.error('Error name:', error.name);
    console.error('Error message:', error.message);

    if (error.name === 'MongoServerSelectionError') {
      console.error('\nPossible causes:');
      console.error('- IP address not whitelisted in Atlas');
      console.error('- Incorrect hostname');
      console.error('- Network/firewall blocking connection');
      console.error('- DNS resolution failed');
    } else if (error.name === 'MongoServerError' && error.codeName === 'AuthenticationFailed') {
      console.error('\nPossible causes:');
      console.error('- Incorrect username or password');
      console.error('- User does not exist');
      console.error('- Incorrect authentication database (authSource)');
    }

    process.exit(1);
  } finally {
    await client.close();
  }
}

// Usage
const uri = process.env.MONGODB_URI || process.argv[2];
if (!uri) {
  console.error('Usage: node connection-test.js <MONGODB_URI>');
  console.error('   or: MONGODB_URI=... node connection-test.js');
  process.exit(1);
}

testConnection(uri);
```

---

## 🚀 Best Practices de Connexion

### Checklist Production

```
┌────────────────────────────────────────────────────────────────────────┐
│              CONNECTION BEST PRACTICES CHECKLIST                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  CONFIGURATION                                                         │
│  ☐ Use SRV connection string (mongodb+srv://)                          │
│  ☐ Configure appropriate connection pool size                          │
│  ☐ Set reasonable timeouts (connect, socket, server selection)         │
│  ☐ Enable retryWrites and retryReads                                   │
│  ☐ Set write concern to "majority" for durability                      │
│  ☐ Configure appropriate read preference                               │
│  ☐ Add application name (appName) for monitoring                       │
│                                                                        │
│  SECURITY                                                              │
│  ☐ Never hardcode credentials                                          │
│  ☐ Use secrets manager (AWS Secrets, Vault, etc.)                      │
│  ☐ Rotate credentials regularly (90 days)                              │
│  ☐ Use TLS/SSL (enabled by default with SRV)                           │
│  ☐ Verify certificates (don't disable in production)                   │
│  ☐ Use least privilege database users                                  │
│  ☐ Consider X.509 certificate authentication                           │
│                                                                        │
│  RELIABILITY                                                           │
│  ☐ Implement connection retry logic                                    │
│  ☐ Handle connection errors gracefully                                 │
│  ☐ Test failover scenarios                                             │
│  ☐ Monitor connection pool metrics                                     │
│  ☐ Set up alerting on connection failures                              │
│  ☐ Implement graceful shutdown                                         │
│                                                                        │
│  PERFORMANCE                                                           │
│  ☐ Reuse connections (don't create per request)                        │
│  ☐ Configure minPoolSize to keep connections warm                      │
│  ☐ Enable compression for bandwidth savings                            │
│  ☐ Use appropriate read preference for workload                        │
│  ☐ Monitor and tune pool size based on metrics                         │
│                                                                        │
│  MONITORING                                                            │
│  ☐ Log connection events (create, close, errors)                       │
│  ☐ Track connection pool utilization                                   │
│  ☐ Monitor connection error rates                                      │
│  ☐ Set up dashboards for connection metrics                            │
│  ☐ Alert on connection pool exhaustion                                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Pattern: Singleton Connection

```javascript
// db.js - Singleton pattern for connection reuse
let cachedClient = null;
let cachedDb = null;

async function connectToDatabase() {
  // Return cached connection if available
  if (cachedClient && cachedDb) {
    return { client: cachedClient, db: cachedDb };
  }

  // Create new connection
  const uri = process.env.MONGODB_URI;
  const client = new MongoClient(uri, {
    maxPoolSize: 50,
    minPoolSize: 10,
    retryWrites: true,
    retryReads: true,
    w: 'majority',
  });

  await client.connect();
  const db = client.db(process.env.DB_NAME || 'mydb');

  // Cache for reuse
  cachedClient = client;
  cachedDb = db;

  return { client, db };
}

module.exports = { connectToDatabase };

// Usage in API route
const { connectToDatabase } = require('./db');

async function handler(req, res) {
  const { db } = await connectToDatabase();
  const users = await db.collection('users').find().toArray();
  res.json(users);
}
```

### Pattern: Graceful Shutdown

```javascript
// app.js - Proper shutdown handling
const { MongoClient } = require('mongodb');
const express = require('express');

const app = express();
const uri = process.env.MONGODB_URI;
const client = new MongoClient(uri);

let server;

async function startServer() {
  try {
    // Connect to MongoDB
    await client.connect();
    console.log('Connected to MongoDB Atlas');

    // Start Express server
    server = app.listen(3000, () => {
      console.log('Server listening on port 3000');
    });

  } catch (error) {
    console.error('Failed to start:', error);
    process.exit(1);
  }
}

async function gracefulShutdown(signal) {
  console.log(`${signal} received, shutting down gracefully...`);

  // Stop accepting new requests
  if (server) {
    server.close(() => {
      console.log('HTTP server closed');
    });
  }

  // Close MongoDB connection
  try {
    await client.close();
    console.log('MongoDB connection closed');
  } catch (error) {
    console.error('Error closing MongoDB:', error);
  }

  process.exit(0);
}

// Handle shutdown signals
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Handle uncaught errors
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection at:', promise, 'reason:', reason);
  gracefulShutdown('unhandledRejection');
});

startServer();
```

---

## 🏁 Résumé

### Points Clés

1. **Connection Strings**
   - ✅ Utiliser SRV format (`mongodb+srv://`)
   - ✅ Configuration via options dans l'URI
   - ✅ Adapter selon l'environnement (dev/staging/prod)

2. **Drivers**
   - Utiliser drivers officiels MongoDB
   - Toujours à jour (dernière version stable)
   - Support natif Atlas (SRV, auto-discovery, failover)

3. **Connection Pooling**
   - Ne PAS créer un client par requête
   - Configurer pool size selon workload
   - Surveiller utilisation du pool

4. **Credentials**
   - ❌ JAMAIS hardcodé
   - ✅ Secrets manager (AWS, Vault, etc.)
   - ✅ Rotation régulière

5. **Troubleshooting**
   - Vérifier IP whitelist
   - Tester DNS resolution
   - Vérifier credentials
   - Utiliser script de diagnostic

### Configuration par Environnement

```javascript
const configs = {
  development: {
    maxPoolSize: 10,
    retryWrites: false,  // See errors immediately
  },

  staging: {
    maxPoolSize: 50,
    minPoolSize: 10,
    retryWrites: true,
    w: 'majority',
  },

  production: {
    maxPoolSize: 100,
    minPoolSize: 20,
    retryWrites: true,
    retryReads: true,
    w: 'majority',
    readPreference: 'primaryPreferred',
    compressors: 'snappy,zlib',
    appName: 'MyApp-Production',
  },
};
```

### Ressources

- [MongoDB Connection String Docs](https://www.mongodb.com/docs/manual/reference/connection-string/)
- [Driver Documentation](https://www.mongodb.com/docs/drivers/)
- [Connection Pooling Guide](https://www.mongodb.com/docs/manual/administration/connection-pool-overview/)

---


⏭️ [Monitoring et alertes dans Atlas](/14-mongodb-atlas/06-monitoring-alertes-atlas.md)
