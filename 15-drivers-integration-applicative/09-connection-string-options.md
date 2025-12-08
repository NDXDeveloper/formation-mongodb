🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.9 Connection String et Options

## Introduction

La **connection string MongoDB** est l'URI qui définit comment les applications se connectent à MongoDB. Elle encode toutes les informations nécessaires : hôtes, authentification, options de connexion, et paramètres de performance. Comprendre sa structure et ses options est crucial pour configurer correctement les applications en production.

## Anatomie d'une Connection String

### Format standard

```
mongodb://[username:password@]host1[:port1][,host2[:port2],...][/database][?options]
```

### Format avec MongoDB+srv (DNS Seedlist)

```
mongodb+srv://[username:password@]host[/database][?options]
```

### Composants

```
mongodb://        ← Protocole (mongodb:// ou mongodb+srv://)
  user:pass       ← Authentification (optionnel)
  @               ← Séparateur auth
  host1:27017     ← Hôte primaire et port
  ,host2:27017    ← Hôtes secondaires (pour replica sets)
  /mydb           ← Database par défaut
  ?               ← Début des options
  retryWrites=true&w=majority  ← Options (key=value&...)
```

## Exemples de base

### Connexion locale simple

```bash
mongodb://localhost:27017
```

### Avec database

```bash
mongodb://localhost:27017/myapp
```

### Avec authentification

```bash
mongodb://admin:SecureP@ss123@localhost:27017/myapp
```

### Replica Set

```bash
mongodb://node1:27017,node2:27017,node3:27017/myapp?replicaSet=rs0
```

### Sharded Cluster (mongos)

```bash
mongodb://mongos1:27017,mongos2:27017/myapp
```

### MongoDB Atlas (DNS Seedlist)

```bash
mongodb+srv://user:password@cluster0.mongodb.net/myapp?retryWrites=true&w=majority
```

## Options d'authentification

### authSource

Spécifie la database pour l'authentification.

```bash
# Authentifier sur 'admin' mais utiliser 'myapp'
mongodb://user:pass@localhost:27017/myapp?authSource=admin
```

**Valeurs communes** :
- `admin` : Database d'authentification par défaut
- Nom de database spécifique

### authMechanism

Mécanisme d'authentification à utiliser.

```bash
mongodb://user:pass@localhost:27017?authMechanism=SCRAM-SHA-256
```

**Valeurs** :
- `SCRAM-SHA-1` : Default pour MongoDB 3.x
- `SCRAM-SHA-256` : Default pour MongoDB 4.x+
- `MONGODB-CR` : Legacy (déprécié)
- `MONGODB-X509` : Certificats X.509
- `GSSAPI` : Kerberos
- `PLAIN` : LDAP
- `MONGODB-AWS` : AWS IAM

### authMechanismProperties

Propriétés additionnelles pour l'authentification.

```bash
# Pour Kerberos
mongodb://user@localhost:27017?authMechanism=GSSAPI&authMechanismProperties=SERVICE_NAME:mongodb

# Pour X.509
mongodb://localhost:27017?authMechanism=MONGODB-X509
```

## Options de connexion

### connectTimeoutMS

Timeout pour établir une connexion (millisecondes).

```bash
mongodb://localhost:27017?connectTimeoutMS=10000
```

**Recommandations** :
- **Développement** : 10000 (10s)
- **Production** : 5000-10000 (5-10s)
- **Mobile/Instable** : 15000-30000 (15-30s)

### socketTimeoutMS

Timeout pour les opérations I/O socket (millisecondes).

```bash
mongodb://localhost:27017?socketTimeoutMS=45000
```

**Recommandations** :
- **Développement** : 0 (pas de timeout)
- **Production** : 30000-60000 (30-60s)
- **Requêtes longues** : 300000+ (5min+)

### serverSelectionTimeoutMS

Timeout pour sélectionner un serveur disponible.

```bash
mongodb://localhost:27017?serverSelectionTimeoutMS=5000
```

**Default** : 30000 (30s)
**Recommandations** : 5000-10000 (5-10s) en production

### heartbeatFrequencyMS

Fréquence des heartbeats pour vérifier l'état des serveurs.

```bash
mongodb://localhost:27017?heartbeatFrequencyMS=10000
```

**Default** : 10000 (10s)
**Minimum** : 500ms

## Options de pool de connexions

### maxPoolSize

Nombre maximum de connexions dans le pool.

```bash
mongodb://localhost:27017?maxPoolSize=100
```

**Recommandations** :
- **Application web** : 50-100
- **Worker/Background** : 10-50
- **Serverless/Lambda** : 1-10
- **Calcul** : CPU_CORES * 2-4

### minPoolSize

Nombre minimum de connexions maintenues.

```bash
mongodb://localhost:27017?minPoolSize=10
```

**Recommandations** :
- **Production** : 10-20
- **Développement** : 0-5

### maxIdleTimeMS

Durée max qu'une connexion peut rester inactive (millisecondes).

```bash
mongodb://localhost:27017?maxIdleTimeMS=60000
```

**Default** : Pas de timeout
**Recommandations** : 60000-300000 (1-5min) en production

### waitQueueTimeoutMS

Timeout pour obtenir une connexion du pool.

```bash
mongodb://localhost:27017?waitQueueTimeoutMS=5000
```

**Default** : 0 (pas de timeout)
**Recommandations** : 5000-10000 (5-10s)

## Options de résilience

### retryWrites

Retry automatique des écritures en cas d'erreur transitoire.

```bash
mongodb://localhost:27017?retryWrites=true
```

**Default** : `true` (MongoDB 4.2+)
**Recommandations** : Toujours `true` en production

### retryReads

Retry automatique des lectures en cas d'erreur transitoire.

```bash
mongodb://localhost:27017?retryReads=true
```

**Default** : `true` (MongoDB 4.4+)
**Recommandations** : Toujours `true` en production

## Options SSL/TLS

### tls / ssl

Activer TLS/SSL.

```bash
mongodb://localhost:27017?tls=true
# ou
mongodb://localhost:27017?ssl=true
```

**Recommandations** :
- **Production** : Toujours `true`
- **Développement local** : `false`

### tlsCAFile / sslCA

Chemin vers le fichier CA.

```bash
mongodb://localhost:27017?tls=true&tlsCAFile=/path/to/ca.pem
```

### tlsCertificateKeyFile / sslCertificateKeyFile

Certificat client.

```bash
mongodb://localhost:27017?tls=true&tlsCertificateKeyFile=/path/to/client.pem
```

### tlsInsecure / sslInsecure

Désactiver la validation du certificat (DANGEREUX).

```bash
mongodb://localhost:27017?tls=true&tlsInsecure=true
```

**⚠️ ATTENTION** : Ne jamais utiliser en production !

### tlsAllowInvalidCertificates

Accepter les certificats invalides (DANGEREUX).

```bash
mongodb://localhost:27017?tls=true&tlsAllowInvalidCertificates=true
```

**⚠️ ATTENTION** : Ne jamais utiliser en production !

### tlsAllowInvalidHostnames

Ignorer la validation du hostname (DANGEREUX).

```bash
mongodb://localhost:27017?tls=true&tlsAllowInvalidHostnames=true
```

**⚠️ ATTENTION** : Seulement pour développement !

## Options de Read Preference

### readPreference

Spécifie où lire les données dans un replica set.

```bash
mongodb://localhost:27017?readPreference=secondary
```

**Valeurs** :
- `primary` : Toujours du primaire (default)
- `primaryPreferred` : Primaire si dispo, sinon secondaire
- `secondary` : Toujours d'un secondaire
- `secondaryPreferred` : Secondaire si dispo, sinon primaire
- `nearest` : Serveur avec la latence la plus faible

### readPreferenceTags

Tags pour filtrer les secondaires.

```bash
mongodb://localhost:27017?readPreference=secondary&readPreferenceTags=dc:west,rack:1
```

**Usage** :
```bash
# Préférer datacenter west
readPreferenceTags=dc:west

# Plusieurs tags (ordre de préférence)
readPreferenceTags=dc:west&readPreferenceTags=dc:east
```

### maxStalenessSeconds

Staleness maximum accepté pour les secondaires.

```bash
mongodb://localhost:27017?readPreference=secondary&maxStalenessSeconds=120
```

**Minimum** : 90 secondes
**Recommandations** : 90-300 (1.5-5min)

## Options de Write Concern

### w (Write Concern)

Nombre d'acknowledgments requis.

```bash
mongodb://localhost:27017?w=majority
```

**Valeurs** :
- `0` : Pas d'acknowledgment (fire and forget)
- `1` : Primaire seulement (default)
- `2`, `3`, etc. : Nombre de nœuds
- `majority` : Majorité des nœuds avec vote
- `<tag_set>` : Tag custom

**Recommandations** :
- **Production** : `majority`
- **Logs/Analytics** : `1`
- **Fire and forget** : `0` (rare)

### journal

Attendre que l'écriture soit journalisée.

```bash
mongodb://localhost:27017?w=1&journal=true
```

**Default** : `true` si w > 0
**Recommandations** : `true` pour données critiques

### wtimeoutMS

Timeout pour Write Concern (millisecondes).

```bash
mongodb://localhost:27017?w=majority&wtimeoutMS=5000
```

**Recommandations** : 5000-10000 (5-10s)

## Options de Read Concern

### readConcernLevel

Niveau de consistance pour les lectures.

```bash
mongodb://localhost:27017?readConcernLevel=majority
```

**Valeurs** :
- `local` : Dernières données (default)
- `available` : Dernières données (même orphelines)
- `majority` : Données confirmées par la majorité
- `linearizable` : Lecture linéarisable (très lent)
- `snapshot` : Snapshot transactionnel

**Recommandations** :
- **Production (critical)** : `majority`
- **Production (standard)** : `local`
- **Analytics** : `available`

## Options de Replica Set

### replicaSet

Nom du replica set.

```bash
mongodb://node1:27017,node2:27017,node3:27017?replicaSet=rs0
```

**Requis** : Pour se connecter à un replica set

### directConnection

Connexion directe à un nœud spécifique.

```bash
mongodb://localhost:27017?directConnection=true
```

**Usage** : Maintenance, debugging
**⚠️ Attention** : Pas de failover automatique

## Options de compression

### compressors

Algorithmes de compression réseau.

```bash
mongodb://localhost:27017?compressors=snappy,zstd,zlib
```

**Valeurs** :
- `snappy` : Rapide, compression moyenne
- `zlib` : Lent, haute compression
- `zstd` : Équilibré (MongoDB 4.2+)

**Recommandations** :
- **Production** : `snappy,zstd`
- **Haute latence** : `zstd,zlib`

### zlibCompressionLevel

Niveau de compression zlib (1-9).

```bash
mongodb://localhost:27017?compressors=zlib&zlibCompressionLevel=6
```

**Default** : 6
**Recommandations** : 4-6 pour équilibre vitesse/compression

## Options diverses

### appName

Nom de l'application pour logging et monitoring.

```bash
mongodb://localhost:27017?appName=MyApp
```

**Recommandations** : Toujours spécifier en production

### serverApi

Version de l'API serveur (Stable API).

```bash
mongodb://localhost:27017?serverApi=1
```

**Valeurs** : `1` (MongoDB 5.0+)

### loadBalanced

Mode load balancer.

```bash
mongodb://localhost:27017?loadBalanced=true
```

**Usage** : Avec MongoDB Atlas ou load balancers

## Exemples par langage

### Node.js / JavaScript

```javascript
// Connection string simple
const uri = 'mongodb://localhost:27017/myapp';

// Production complète
const uri = [
  'mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp',
  '?replicaSet=rs0',
  '&authSource=admin',
  '&retryWrites=true',
  '&retryReads=true',
  '&w=majority',
  '&readPreference=primaryPreferred',
  '&maxPoolSize=100',
  '&minPoolSize=10',
  '&maxIdleTimeMS=60000',
  '&serverSelectionTimeoutMS=5000',
  '&socketTimeoutMS=45000',
  '&connectTimeoutMS=10000',
  '&compressors=snappy,zstd',
  '&appName=MyApp'
].join('');

// MongoDB+srv avec options
const uri = [
  'mongodb+srv://user:pass@cluster0.mongodb.net/myapp',
  '?retryWrites=true',
  '&w=majority',
  '&readPreference=secondaryPreferred',
  '&maxPoolSize=50'
].join('');

// Options programmatiques
import { MongoClient } from 'mongodb';

const client = new MongoClient(uri, {
  maxPoolSize: 100,
  minPoolSize: 10,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  family: 4, // Force IPv4
  compressors: ['snappy', 'zstd']
});
```

### Python

```python
from pymongo import MongoClient

# Connection string simple
uri = "mongodb://localhost:27017/myapp"

# Production complète
uri = (
    "mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp"
    "?replicaSet=rs0"
    "&authSource=admin"
    "&retryWrites=true"
    "&retryReads=true"
    "&w=majority"
    "&readPreference=primaryPreferred"
    "&maxPoolSize=100"
    "&minPoolSize=10"
    "&maxIdleTimeMS=60000"
    "&serverSelectionTimeoutMS=5000"
    "&socketTimeoutMS=45000"
    "&connectTimeoutMS=10000"
    "&compressors=snappy,zstd"
    "&appName=MyApp"
)

# Options programmatiques
client = MongoClient(
    uri,
    maxPoolSize=100,
    minPoolSize=10,
    serverSelectionTimeoutMS=5000,
    socketTimeoutMS=45000,
    connectTimeoutMS=10000,
    compressors=['snappy', 'zstd'],
    appname='MyApp'
)
```

### Java

```java
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import java.util.concurrent.TimeUnit;

// Connection string simple
String uri = "mongodb://localhost:27017/myapp";

// Production complète
String uri = "mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp"
    + "?replicaSet=rs0"
    + "&authSource=admin"
    + "&retryWrites=true"
    + "&retryReads=true"
    + "&w=majority"
    + "&readPreference=primaryPreferred"
    + "&maxPoolSize=100"
    + "&minPoolSize=10"
    + "&maxIdleTimeMS=60000"
    + "&serverSelectionTimeoutMS=5000"
    + "&socketTimeoutMS=45000"
    + "&connectTimeoutMS=10000"
    + "&compressors=snappy,zstd"
    + "&appName=MyApp";

// Options programmatiques
ConnectionString connString = new ConnectionString(uri);
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(connString)
    .applyToConnectionPoolSettings(builder ->
        builder.maxSize(100)
               .minSize(10)
               .maxConnectionIdleTime(60, TimeUnit.SECONDS)
               .maxWaitTime(5, TimeUnit.SECONDS))
    .applyToSocketSettings(builder ->
        builder.connectTimeout(10, TimeUnit.SECONDS)
               .readTimeout(45, TimeUnit.SECONDS))
    .applyToClusterSettings(builder ->
        builder.serverSelectionTimeout(5, TimeUnit.SECONDS))
    .compressorList(Arrays.asList(
        MongoCompressor.createSnappyCompressor(),
        MongoCompressor.createZstdCompressor()))
    .applicationName("MyApp")
    .build();

MongoClient client = MongoClients.create(settings);
```

### C# / .NET

```csharp
using MongoDB.Driver;

// Connection string simple
var uri = "mongodb://localhost:27017/myapp";

// Production complète
var uri = "mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp"
    + "?replicaSet=rs0"
    + "&authSource=admin"
    + "&retryWrites=true"
    + "&retryReads=true"
    + "&w=majority"
    + "&readPreference=primaryPreferred"
    + "&maxPoolSize=100"
    + "&minPoolSize=10"
    + "&maxIdleTimeMS=60000"
    + "&serverSelectionTimeoutMS=5000"
    + "&socketTimeoutMS=45000"
    + "&connectTimeoutMS=10000"
    + "&compressors=snappy,zstd"
    + "&appName=MyApp";

// Options programmatiques
var settings = MongoClientSettings.FromConnectionString(uri);
settings.MaxConnectionPoolSize = 100;
settings.MinConnectionPoolSize = 10;
settings.MaxConnectionIdleTime = TimeSpan.FromMinutes(1);
settings.ServerSelectionTimeout = TimeSpan.FromSeconds(5);
settings.SocketTimeout = TimeSpan.FromSeconds(45);
settings.ConnectTimeout = TimeSpan.FromSeconds(10);
settings.Compressors = new[] {
    new CompressorConfiguration(CompressorType.Snappy),
    new CompressorConfiguration(CompressorType.Zstd)
};
settings.ApplicationName = "MyApp";

var client = new MongoClient(settings);
```

### Go

```go
package main

import (
    "context"
    "time"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

// Connection string simple
uri := "mongodb://localhost:27017/myapp"

// Production complète
uri := "mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp" +
    "?replicaSet=rs0" +
    "&authSource=admin" +
    "&retryWrites=true" +
    "&retryReads=true" +
    "&w=majority" +
    "&readPreference=primaryPreferred" +
    "&maxPoolSize=100" +
    "&minPoolSize=10" +
    "&maxIdleTimeMS=60000" +
    "&serverSelectionTimeoutMS=5000" +
    "&socketTimeoutMS=45000" +
    "&connectTimeoutMS=10000" +
    "&compressors=snappy,zstd" +
    "&appName=MyApp"

// Options programmatiques
clientOpts := options.Client().
    ApplyURI(uri).
    SetMaxPoolSize(100).
    SetMinPoolSize(10).
    SetMaxConnIdleTime(60 * time.Second).
    SetServerSelectionTimeout(5 * time.Second).
    SetSocketTimeout(45 * time.Second).
    SetConnectTimeout(10 * time.Second).
    SetCompressors([]string{"snappy", "zstd"}).
    SetAppName("MyApp")

client, err := mongo.Connect(context.Background(), clientOpts)
```

### PHP

```php
<?php
use MongoDB\Client;

// Connection string simple
$uri = 'mongodb://localhost:27017/myapp';

// Production complète
$uri = 'mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp' .
    '?replicaSet=rs0' .
    '&authSource=admin' .
    '&retryWrites=true' .
    '&retryReads=true' .
    '&w=majority' .
    '&readPreference=primaryPreferred' .
    '&maxPoolSize=100' .
    '&minPoolSize=10' .
    '&maxIdleTimeMS=60000' .
    '&serverSelectionTimeoutMS=5000' .
    '&socketTimeoutMS=45000' .
    '&connectTimeoutMS=10000' .
    '&compressors=snappy,zstd' .
    '&appName=MyApp';

// Options programmatiques
$client = new Client($uri, [
    'maxPoolSize' => 100,
    'minPoolSize' => 10,
    'serverSelectionTimeoutMS' => 5000,
    'socketTimeoutMS' => 45000,
    'connectTimeoutMS' => 10000,
    'compressors' => ['snappy', 'zstd'],
    'appname' => 'MyApp'
]);
```

### Ruby

```ruby
require 'mongo'

# Connection string simple
uri = 'mongodb://localhost:27017/myapp'

# Production complète
uri = 'mongodb://user:pass@node1:27017,node2:27017,node3:27017/myapp' \
      '?replicaSet=rs0' \
      '&authSource=admin' \
      '&retryWrites=true' \
      '&retryReads=true' \
      '&w=majority' \
      '&readPreference=primaryPreferred' \
      '&maxPoolSize=100' \
      '&minPoolSize=10' \
      '&maxIdleTimeMS=60000' \
      '&serverSelectionTimeoutMS=5000' \
      '&socketTimeoutMS=45000' \
      '&connectTimeoutMS=10000' \
      '&compressors=snappy,zstd' \
      '&appName=MyApp'

# Options programmatiques
client = Mongo::Client.new(uri, {
  max_pool_size: 100,
  min_pool_size: 10,
  max_idle_time: 60,
  server_selection_timeout: 5,
  socket_timeout: 45,
  connect_timeout: 10,
  compressors: ['snappy', 'zstd'],
  app_name: 'MyApp'
})
```

## Configuration par environnement

### Variables d'environnement

```bash
# .env.development
MONGODB_URI=mongodb://localhost:27017/myapp_dev?retryWrites=true

# .env.staging
MONGODB_URI=mongodb://user:pass@staging-db:27017/myapp_staging?replicaSet=rs0&w=majority&retryWrites=true

# .env.production
MONGODB_URI=mongodb+srv://prod_user:SecurePass@cluster0.mongodb.net/myapp_prod?retryWrites=true&w=majority&readPreference=primaryPreferred&maxPoolSize=100&minPoolSize=20&compressors=snappy,zstd&appName=MyApp
```

### Configuration par profil

```javascript
// config/database.js
const configs = {
  development: {
    uri: 'mongodb://localhost:27017/myapp_dev',
    options: {
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000
    }
  },

  test: {
    uri: 'mongodb://localhost:27017/myapp_test',
    options: {
      maxPoolSize: 5,
      serverSelectionTimeoutMS: 2000
    }
  },

  staging: {
    uri: process.env.MONGODB_URI,
    options: {
      maxPoolSize: 50,
      minPoolSize: 10,
      retryWrites: true,
      retryReads: true,
      w: 'majority',
      serverSelectionTimeoutMS: 5000
    }
  },

  production: {
    uri: process.env.MONGODB_URI,
    options: {
      maxPoolSize: 100,
      minPoolSize: 20,
      maxIdleTimeMS: 60000,
      retryWrites: true,
      retryReads: true,
      w: 'majority',
      readPreference: 'primaryPreferred',
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
      connectTimeoutMS: 10000,
      compressors: ['snappy', 'zstd'],
      appName: 'MyApp'
    }
  }
};

const env = process.env.NODE_ENV || 'development';
module.exports = configs[env];
```

## Bonnes pratiques

### ✅ DO - À faire

```bash
# 1. Utiliser mongodb+srv:// pour Atlas
mongodb+srv://user:pass@cluster0.mongodb.net/myapp

# 2. Toujours spécifier retryWrites et retryReads
?retryWrites=true&retryReads=true

# 3. Utiliser w=majority en production
?w=majority&wtimeoutMS=5000

# 4. Configurer les timeouts appropriés
?connectTimeoutMS=10000&socketTimeoutMS=45000&serverSelectionTimeoutMS=5000

# 5. Activer la compression
?compressors=snappy,zstd

# 6. Spécifier un appName
?appName=MyApp

# 7. Utiliser TLS en production
?tls=true

# 8. Configurer le pool de connexions
?maxPoolSize=100&minPoolSize=10

# 9. Encoder les caractères spéciaux dans les mots de passe
# Utiliser encodeURIComponent() ou équivalent
const password = 'p@ss/word!';
const encoded = encodeURIComponent(password); // 'p%40ss%2Fword%21'

# 10. Stocker les URI dans des variables d'environnement
# Jamais dans le code source !
```

### ❌ DON'T - À éviter

```bash
# ❌ Ne pas hardcoder les credentials
mongodb://admin:password123@localhost:27017

# ❌ Ne pas désactiver TLS en production
?tls=false

# ❌ Ne pas utiliser tlsInsecure en production
?tls=true&tlsInsecure=true

# ❌ Ne pas utiliser w=0 pour données critiques
?w=0

# ❌ Ne pas ignorer les timeouts
# Toujours définir des timeouts appropriés

# ❌ Ne pas utiliser maxPoolSize trop élevé
?maxPoolSize=1000  # Trop élevé !

# ❌ Ne pas utiliser directConnection avec replica sets
?directConnection=true&replicaSet=rs0  # Incohérent !

# ❌ Ne pas mélanger mongodb:// et mongodb+srv://
# Ce sont des formats différents

# ❌ Ne pas oublier d'encoder les caractères spéciaux
mongodb://user:p@ssword@host  # '@' dans le mot de passe !

# ❌ Ne pas utiliser readPreference=secondary pour writes
# Les writes vont toujours au primaire
```

## Sécurité

### Encoder les caractères spéciaux

```javascript
// JavaScript
function buildURI(user, password, host, database) {
  const encodedUser = encodeURIComponent(user);
  const encodedPass = encodeURIComponent(password);
  return `mongodb://${encodedUser}:${encodedPass}@${host}/${database}`;
}

// Exemple
const uri = buildURI('admin', 'p@ss/word!', 'localhost:27017', 'myapp');
// mongodb://admin:p%40ss%2Fword%21@localhost:27017/myapp
```

```python
# Python
from urllib.parse import quote_plus

def build_uri(user, password, host, database):
    encoded_user = quote_plus(user)
    encoded_pass = quote_plus(password)
    return f"mongodb://{encoded_user}:{encoded_pass}@{host}/{database}"

# Exemple
uri = build_uri('admin', 'p@ss/word!', 'localhost:27017', 'myapp')
# mongodb://admin:p%40ss%2Fword%21@localhost:27017/myapp
```

### Masquer les credentials dans les logs

```javascript
// Ne jamais logger l'URI complète !
function sanitizeURI(uri) {
  return uri.replace(/mongodb(\+srv)?:\/\/[^@]+@/, 'mongodb$1://***:***@');
}

console.log(sanitizeURI(uri));
// mongodb://***:***@localhost:27017/myapp?...
```

### Utiliser des secrets managers

```javascript
// AWS Secrets Manager
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

async function getMongoURI() {
  const secret = await secretsManager.getSecretValue({
    SecretId: 'prod/mongodb/uri'
  }).promise();

  return JSON.parse(secret.SecretString).uri;
}
```

## Résolution de problèmes

### Problème : Connection timeout

```bash
# Symptôme
MongoServerSelectionError: Server selection timed out after 30000 ms

# Solutions
1. Vérifier que MongoDB est accessible (firewall, network)
2. Augmenter serverSelectionTimeoutMS
   ?serverSelectionTimeoutMS=10000
3. Vérifier les paramètres réseau (DNS, routing)
4. Pour replica sets, vérifier que tous les nœuds sont accessibles
```

### Problème : Authentication failed

```bash
# Symptôme
MongoError: Authentication failed

# Solutions
1. Vérifier les credentials
2. Spécifier authSource si différent de la database
   ?authSource=admin
3. Vérifier le mécanisme d'authentification
   ?authMechanism=SCRAM-SHA-256
4. S'assurer que l'utilisateur existe dans la bonne database
   use admin
   db.createUser({user: "myuser", pwd: "mypass", roles: [{role: "readWrite", db: "myapp"}]})
```

### Problème : Pool exhausted

```bash
# Symptôme
MongoWaitQueueFullError: Too many operations waiting for connection

# Solutions
1. Augmenter maxPoolSize
   ?maxPoolSize=200
2. Réduire les opérations parallèles
3. Vérifier les connexions qui ne sont pas fermées
4. Ajouter waitQueueTimeoutMS pour fail fast
   ?waitQueueTimeoutMS=5000
5. Vérifier les slow queries qui bloquent des connexions
```

### Problème : TLS/SSL errors

```bash
# Symptôme
MongoError: SSL routines:tls_process_server_certificate:certificate verify failed

# Solutions
1. Vérifier que le certificat CA est correct
   ?tls=true&tlsCAFile=/path/to/ca.pem
2. Pour développement uniquement, désactiver validation
   ?tls=true&tlsAllowInvalidCertificates=true
3. Vérifier la date d'expiration du certificat
4. S'assurer que le hostname correspond au certificat
```

### Problème : Slow queries

```bash
# Symptôme
Operations taking too long

# Solutions
1. Augmenter socketTimeoutMS
   ?socketTimeoutMS=120000
2. Vérifier les index
   db.collection.find({...}).explain()
3. Optimiser les queries
4. Utiliser des projections
5. Paginer les résultats
```

## Configuration recommandée par scénario

### Application Web Standard

```bash
mongodb://user:pass@host1,host2,host3/myapp?
  replicaSet=rs0&
  authSource=admin&
  retryWrites=true&
  retryReads=true&
  w=majority&
  wtimeoutMS=5000&
  readPreference=primaryPreferred&
  maxPoolSize=100&
  minPoolSize=10&
  maxIdleTimeMS=60000&
  serverSelectionTimeoutMS=5000&
  socketTimeoutMS=45000&
  connectTimeoutMS=10000&
  compressors=snappy,zstd&
  appName=WebApp
```

### Microservices

```bash
mongodb://user:pass@host1,host2,host3/myapp?
  replicaSet=rs0&
  authSource=admin&
  retryWrites=true&
  retryReads=true&
  w=1&
  readPreference=nearest&
  maxPoolSize=50&
  minPoolSize=5&
  serverSelectionTimeoutMS=3000&
  socketTimeoutMS=30000&
  connectTimeoutMS=5000&
  compressors=snappy&
  appName=ServiceName
```

### Background Workers

```bash
mongodb://user:pass@host1,host2,host3/myapp?
  replicaSet=rs0&
  authSource=admin&
  retryWrites=true&
  w=1&
  maxPoolSize=20&
  minPoolSize=5&
  socketTimeoutMS=300000&
  connectTimeoutMS=10000&
  appName=Worker
```

### Analytics / Reporting

```bash
mongodb://user:pass@host1,host2,host3/myapp?
  replicaSet=rs0&
  authSource=admin&
  retryReads=true&
  readPreference=secondary&
  readConcernLevel=available&
  maxPoolSize=30&
  socketTimeoutMS=600000&
  connectTimeoutMS=15000&
  compressors=zstd,zlib&
  appName=Analytics
```

### Serverless / Lambda

```bash
mongodb://user:pass@host/myapp?
  retryWrites=true&
  retryReads=true&
  w=majority&
  maxPoolSize=10&
  minPoolSize=1&
  maxIdleTimeMS=10000&
  serverSelectionTimeoutMS=5000&
  socketTimeoutMS=20000&
  connectTimeoutMS=5000&
  appName=Lambda
```

## Tableau récapitulatif des options importantes

| Option | Default | Recommandation Production | Description |
|--------|---------|---------------------------|-------------|
| **maxPoolSize** | 100 | 50-100 (web), 20-50 (worker) | Connexions max dans le pool |
| **minPoolSize** | 0 | 10-20 | Connexions min maintenues |
| **connectTimeoutMS** | 10000 | 10000 | Timeout connexion initiale |
| **socketTimeoutMS** | 0 | 30000-60000 | Timeout I/O socket |
| **serverSelectionTimeoutMS** | 30000 | 5000 | Timeout sélection serveur |
| **retryWrites** | true | true | Retry auto des writes |
| **retryReads** | true | true | Retry auto des reads |
| **w** | 1 | majority | Write concern |
| **wtimeoutMS** | - | 5000 | Timeout write concern |
| **readPreference** | primary | primaryPreferred | Où lire |
| **readConcernLevel** | local | local ou majority | Niveau consistance |
| **compressors** | - | snappy,zstd | Compression réseau |
| **tls** | false | true | Activer TLS |
| **appName** | - | YourAppName | Nom pour monitoring |

## Conclusion

La connection string MongoDB est le point d'entrée pour configurer correctement vos applications. Les points clés à retenir :

1. **Sécurité** : TLS activé, credentials encodés, secrets externalisés
2. **Résilience** : retryWrites/retryReads activés, timeouts appropriés
3. **Performance** : Pool dimensionné, compression activée, read preference optimisée
4. **Monitoring** : appName défini pour traçabilité
5. **Environnement** : Configuration adaptée par environnement

Une configuration appropriée garantit des applications robustes et performantes en production.

---


⏭️ [Connection Pooling](/15-drivers-integration-applicative/10-connection-pooling.md)
