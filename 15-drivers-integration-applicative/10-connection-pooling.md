🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.10 Connection Pooling

## Introduction

Le **connection pooling** est un mécanisme essentiel pour optimiser les performances des applications utilisant MongoDB. Au lieu de créer et fermer une connexion pour chaque opération, un pool maintient un ensemble de connexions réutilisables. Comprendre comment configurer et gérer efficacement ce pool est crucial pour la scalabilité et la performance en production.

## Concepts fondamentaux

### Qu'est-ce qu'un connection pool ?

Un connection pool est un cache de connexions réseau maintenues ouvertes et réutilisables. Quand une application a besoin d'effectuer une opération, elle :

1. **Emprunte** une connexion du pool
2. **Utilise** la connexion pour l'opération
3. **Retourne** la connexion au pool (pas de fermeture)

### Avantages du pooling

```
Sans pool :                     Avec pool :
┌──────────────┐               ┌───────────────┐
│  Operation 1 │               │  Operation 1  │
│  Create conn │               │  Get from pool│
│  Use conn    │               │  Use conn     │
│  Close conn  │  ← Coûteux    │  Return pool  │  ← Rapide
└──────────────┘               └───────────────┘
      150ms                          2ms

┌──────────────┐               ┌───────────────┐
│  Operation 2 │               │  Operation 2  │
│  Create conn │               │  Get from pool│
│  Use conn    │               │  Use conn     │
│  Close conn  │  ← Coûteux    │  Return pool  │  ← Rapide
└──────────────┘               └───────────────┘
      150ms                          2ms
```

**Bénéfices** :
- ✅ Réduction drastique de la latence
- ✅ Diminution de la charge CPU (client et serveur)
- ✅ Meilleure utilisation des ressources
- ✅ Scalabilité améliorée
- ✅ Réduction du handshake SSL/TLS répété

### Anatomie d'un pool MongoDB

```
┌─────────────────────────────────────────────┐
│          Connection Pool                    │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  CONN 1  │  │  CONN 2  │  │  CONN 3  │   │ ← Active
│  │  IN USE  │  │  IN USE  │  │  IN USE  │   │   (en utilisation)
│  └──────────┘  └──────────┘  └──────────┘   │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  CONN 4  │  │  CONN 5  │  │  CONN 6  │   │ ← Available
│  │  IDLE    │  │  IDLE    │  │  IDLE    │   │   (disponibles)
│  └──────────┘  └──────────┘  └──────────┘   │
│                                             │
│  minPoolSize: 3  ←──────────┐               │
│  maxPoolSize: 10 ←──────┐   │               │
│  Current: 6             │   │               │
│                         │   │               │
│  ┌───────────────────┐  │   │               │
│  │   Wait Queue      │  │   │               │
│  │  Request 1...     │  │   │               │
│  │  Request 2...     │◄─┘   │               │
│  └───────────────────┘      │               │
│  maxPoolSize reached ───────┘               │
└─────────────────────────────────────────────┘
```

## Configuration du pool

### Paramètres clés

#### maxPoolSize

Nombre maximum de connexions dans le pool.

```javascript
// Node.js
const client = new MongoClient(uri, {
  maxPoolSize: 100
});
```

**Facteurs de dimensionnement** :
- Nombre de threads/workers de l'application
- Charge concurrente attendue
- Ressources du serveur MongoDB
- Limites réseau

**Formule générale** :
```
maxPoolSize = (Concurrent Requests × Average Query Time) / 1000ms
```

**Exemples** :
- **Web app (100 req/s, 50ms query)** : 100 × 50 / 1000 = 5 (minimum 10-20)
- **API intensive (500 req/s, 100ms query)** : 500 × 100 / 1000 = 50
- **Worker (10 jobs, 2s query)** : 10 × 2000 / 1000 = 20

#### minPoolSize

Nombre minimum de connexions maintenues.

```javascript
// Node.js
const client = new MongoClient(uri, {
  minPoolSize: 10
});
```

**Recommandations** :
- **Production** : 10-20% de maxPoolSize
- **Développement** : 0-5
- **Serverless** : 0-1 (cold starts)

**Avantages** :
- Connexions prêtes pour pics de charge
- Réduction du cold start
- Warm pool constant

#### maxIdleTimeMS

Durée max qu'une connexion peut rester inactive.

```javascript
// Node.js
const client = new MongoClient(uri, {
  maxIdleTimeMS: 60000 // 60 secondes
});
```

**Recommandations** :
- **Production** : 60000-300000 (1-5min)
- **Serverless** : 10000-30000 (10-30s)
- **Long-running** : 300000+ (5min+)

#### waitQueueTimeoutMS

Timeout pour obtenir une connexion du pool.

```javascript
// Node.js
const client = new MongoClient(uri, {
  waitQueueTimeoutMS: 5000 // 5 secondes
});
```

**Recommandations** :
- **Web API** : 2000-5000 (2-5s)
- **Background jobs** : 10000-30000 (10-30s)
- **0** : Attendre indéfiniment (dangereux)

### Configuration complète par langage

#### Node.js / JavaScript

```javascript
import { MongoClient } from 'mongodb';

const uri = process.env.MONGODB_URI;

// Configuration optimale pour application web
const client = new MongoClient(uri, {
  // Pool configuration
  maxPoolSize: 100,           // Max connexions
  minPoolSize: 10,            // Min connexions maintenues
  maxIdleTimeMS: 60000,       // 1 minute idle max
  waitQueueTimeoutMS: 5000,   // 5s timeout pour obtenir conn

  // Connection timeouts
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000,
  serverSelectionTimeoutMS: 5000,

  // Monitoring
  monitorCommands: true
});

// Event listeners pour monitoring
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

client.on('connectionPoolCleared', (event) => {
  console.log('Pool cleared');
});

client.on('connectionPoolClosed', (event) => {
  console.log('Pool closed');
});

await client.connect();
```

#### Python (PyMongo)

```python
from pymongo import MongoClient
from pymongo.monitoring import (
    ConnectionPoolListener,
    PoolCreatedEvent,
    ConnectionCreatedEvent,
    ConnectionReadyEvent,
    ConnectionCheckedOutEvent,
    ConnectionCheckedInEvent,
    ConnectionClosedEvent
)
import logging

# Listener pour monitoring
class PoolMonitor(ConnectionPoolListener):
    def pool_created(self, event: PoolCreatedEvent):
        logging.info(f"Pool created: {event.options}")

    def connection_created(self, event: ConnectionCreatedEvent):
        logging.info(f"Connection created: {event.connection_id}")

    def connection_ready(self, event: ConnectionReadyEvent):
        logging.info(f"Connection ready: {event.connection_id}")

    def connection_checked_out(self, event: ConnectionCheckedOutEvent):
        logging.info(f"Connection checked out: {event.connection_id}")

    def connection_checked_in(self, event: ConnectionCheckedInEvent):
        logging.info(f"Connection checked in: {event.connection_id}")

    def connection_closed(self, event: ConnectionClosedEvent):
        logging.info(f"Connection closed: {event.connection_id}, reason: {event.reason}")

# Configuration
uri = os.getenv('MONGODB_URI')

client = MongoClient(
    uri,
    # Pool configuration
    maxPoolSize=100,
    minPoolSize=10,
    maxIdleTimeMS=60000,
    waitQueueTimeoutMS=5000,

    # Connection timeouts
    connectTimeoutMS=10000,
    socketTimeoutMS=45000,
    serverSelectionTimeoutMS=5000,

    # Monitoring
    event_listeners=[PoolMonitor()]
)
```

#### Java

```java
import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.connection.*;
import com.mongodb.event.*;
import java.util.concurrent.TimeUnit;

public class MongoDBPoolConfig {
    public static MongoClient createClient() {
        String uri = System.getenv("MONGODB_URI");
        ConnectionString connString = new ConnectionString(uri);

        // Pool listener pour monitoring
        ConnectionPoolListener poolListener = new ConnectionPoolListener() {
            @Override
            public void connectionPoolCreated(ConnectionPoolCreatedEvent event) {
                System.out.println("Pool created: " + event.getServerId());
            }

            @Override
            public void connectionCreated(ConnectionCreatedEvent event) {
                System.out.println("Connection created: " + event.getConnectionId());
            }

            @Override
            public void connectionReady(ConnectionReadyEvent event) {
                System.out.println("Connection ready: " + event.getConnectionId());
            }

            @Override
            public void connectionCheckedOut(ConnectionCheckedOutEvent event) {
                System.out.println("Connection checked out: " + event.getConnectionId());
            }

            @Override
            public void connectionCheckedIn(ConnectionCheckedInEvent event) {
                System.out.println("Connection checked in: " + event.getConnectionId());
            }

            @Override
            public void connectionClosed(ConnectionClosedEvent event) {
                System.out.println("Connection closed: " + event.getConnectionId()
                    + ", reason: " + event.getReason());
            }
        };

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connString)
            .applyToConnectionPoolSettings(builder ->
                builder
                    .maxSize(100)
                    .minSize(10)
                    .maxConnectionIdleTime(60, TimeUnit.SECONDS)
                    .maxWaitTime(5, TimeUnit.SECONDS)
                    .addConnectionPoolListener(poolListener))
            .applyToSocketSettings(builder ->
                builder
                    .connectTimeout(10, TimeUnit.SECONDS)
                    .readTimeout(45, TimeUnit.SECONDS))
            .applyToClusterSettings(builder ->
                builder
                    .serverSelectionTimeout(5, TimeUnit.SECONDS))
            .build();

        return MongoClients.create(settings);
    }
}
```

#### C# / .NET

```csharp
using MongoDB.Driver;
using MongoDB.Driver.Core.Events;
using System;

public class MongoDBPoolConfig
{
    public static MongoClient CreateClient()
    {
        var uri = Environment.GetEnvironmentVariable("MONGODB_URI");
        var settings = MongoClientSettings.FromConnectionString(uri);

        // Pool configuration
        settings.MaxConnectionPoolSize = 100;
        settings.MinConnectionPoolSize = 10;
        settings.MaxConnectionIdleTime = TimeSpan.FromMinutes(1);
        settings.WaitQueueTimeout = TimeSpan.FromSeconds(5);

        // Connection timeouts
        settings.ConnectTimeout = TimeSpan.FromSeconds(10);
        settings.SocketTimeout = TimeSpan.FromSeconds(45);
        settings.ServerSelectionTimeout = TimeSpan.FromSeconds(5);

        // Monitoring
        settings.ClusterConfigurator = cb =>
        {
            cb.Subscribe<ConnectionPoolOpenedEvent>(e =>
                Console.WriteLine($"Pool opened: {e.ServerId}"));

            cb.Subscribe<ConnectionPoolClosedEvent>(e =>
                Console.WriteLine($"Pool closed: {e.ServerId}"));

            cb.Subscribe<ConnectionCreatedEvent>(e =>
                Console.WriteLine($"Connection created: {e.ConnectionId}"));

            cb.Subscribe<ConnectionOpenedEvent>(e =>
                Console.WriteLine($"Connection opened: {e.ConnectionId}"));

            cb.Subscribe<ConnectionPoolCheckedOutConnectionEvent>(e =>
                Console.WriteLine($"Connection checked out: {e.ConnectionId}"));

            cb.Subscribe<ConnectionPoolCheckedInConnectionEvent>(e =>
                Console.WriteLine($"Connection checked in: {e.ConnectionId}"));

            cb.Subscribe<ConnectionClosedEvent>(e =>
                Console.WriteLine($"Connection closed: {e.ConnectionId}"));
        };

        return new MongoClient(settings);
    }
}
```

#### Go

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "time"

    "go.mongodb.org/mongo-driver/event"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

// PoolMonitor pour surveiller le pool
type PoolMonitor struct{}

func (m *PoolMonitor) PoolEvent(e *event.PoolEvent) {
    switch e.Type {
    case event.PoolCreated:
        log.Printf("Pool created")
    case event.PoolCleared:
        log.Printf("Pool cleared")
    case event.PoolClosedEvent:
        log.Printf("Pool closed")
    case event.ConnectionCreated:
        log.Printf("Connection created: %d", e.ConnectionID)
    case event.ConnectionReady:
        log.Printf("Connection ready: %d", e.ConnectionID)
    case event.ConnectionClosed:
        log.Printf("Connection closed: %d, reason: %s", e.ConnectionID, e.Reason)
    case event.ConnectionCheckOut:
        log.Printf("Connection check out: %d", e.ConnectionID)
    case event.ConnectionCheckedOut:
        log.Printf("Connection checked out: %d", e.ConnectionID)
    case event.ConnectionCheckedIn:
        log.Printf("Connection checked in: %d", e.ConnectionID)
    }
}

func createClient() (*mongo.Client, error) {
    uri := os.Getenv("MONGODB_URI")

    // Pool monitor
    poolMonitor := &event.PoolMonitor{
        Event: func(e *event.PoolEvent) {
            (&PoolMonitor{}).PoolEvent(e)
        },
    }

    clientOpts := options.Client().
        ApplyURI(uri).
        SetMaxPoolSize(100).
        SetMinPoolSize(10).
        SetMaxConnIdleTime(60 * time.Second).
        SetConnectTimeout(10 * time.Second).
        SetSocketTimeout(45 * time.Second).
        SetServerSelectionTimeout(5 * time.Second).
        SetPoolMonitor(poolMonitor)

    client, err := mongo.Connect(context.Background(), clientOpts)
    if err != nil {
        return nil, fmt.Errorf("failed to connect: %w", err)
    }

    return client, nil
}
```

#### PHP

```php
<?php
use MongoDB\Client;
use MongoDB\Driver\Monitoring\CommandSubscriber;
use MongoDB\Driver\Monitoring\CommandStartedEvent;
use MongoDB\Driver\Monitoring\CommandSucceededEvent;

// Monitoring (approximatif, PHP n'a pas d'events pool directs)
class ConnectionMonitor implements CommandSubscriber
{
    private $activeConnections = 0;

    public function commandStarted(CommandStartedEvent $event): void
    {
        $this->activeConnections++;
        error_log("Active connections: {$this->activeConnections}");
    }

    public function commandSucceeded(CommandSucceededEvent $event): void
    {
        $this->activeConnections--;
    }

    public function commandFailed(CommandFailedEvent $event): void
    {
        $this->activeConnections--;
    }
}

$uri = getenv('MONGODB_URI');

$client = new Client($uri, [
    // Pool configuration
    'maxPoolSize' => 100,
    'minPoolSize' => 10,
    'maxIdleTimeMS' => 60000,
    'waitQueueTimeoutMS' => 5000,

    // Connection timeouts
    'connectTimeoutMS' => 10000,
    'socketTimeoutMS' => 45000,
    'serverSelectionTimeoutMS' => 5000
]);

// Ajouter monitoring
$monitor = new ConnectionMonitor();
\MongoDB\Driver\Monitoring\addSubscriber($monitor);
```

#### Ruby

```ruby
require 'mongo'

# Listener pour monitoring
class PoolListener
  def published(event)
    case event
    when Mongo::Monitoring::Event::ConnectionPoolCreated
      puts "Pool created"
    when Mongo::Monitoring::Event::ConnectionCreated
      puts "Connection created: #{event.connection_id}"
    when Mongo::Monitoring::Event::ConnectionReady
      puts "Connection ready: #{event.connection_id}"
    when Mongo::Monitoring::Event::ConnectionCheckOut
      puts "Connection check out: #{event.connection_id}"
    when Mongo::Monitoring::Event::ConnectionCheckedOut
      puts "Connection checked out: #{event.connection_id}"
    when Mongo::Monitoring::Event::ConnectionCheckedIn
      puts "Connection checked in: #{event.connection_id}"
    when Mongo::Monitoring::Event::ConnectionClosed
      puts "Connection closed: #{event.connection_id}, reason: #{event.reason}"
    end
  end
end

uri = ENV['MONGODB_URI']

client = Mongo::Client.new(uri, {
  # Pool configuration
  max_pool_size: 100,
  min_pool_size: 10,
  max_idle_time: 60,  # secondes
  wait_queue_timeout: 5,  # secondes

  # Connection timeouts
  connect_timeout: 10,
  socket_timeout: 45,
  server_selection_timeout: 5
})

# Ajouter monitoring
listener = PoolListener.new
Mongo::Monitoring::Global.subscribe(
  Mongo::Monitoring::CONNECTION_POOL,
  listener
)
```

## Dimensionnement du pool

### Formules et stratégies

#### Calcul de base

```
maxPoolSize = max(
  Expected Concurrent Operations,
  (Peak Requests/sec × Average Query Time in seconds),
  Number of Application Threads
)
```

#### Facteurs à considérer

```
┌─────────────────────────────────────────────┐
│  Facteurs influençant le dimensionnement    │
├─────────────────────────────────────────────┤
│  1. Charge applicative                      │
│     - Requêtes/seconde                      │
│     - Pics de charge                        │
│     - Patterns d'utilisation                │
│                                             │
│  2. Performance des requêtes                │
│     - Temps moyen de requête                │
│     - Slow queries                          │
│     - Aggregations complexes                │
│                                             │
│  3. Architecture applicative                │
│     - Nombre de workers/threads             │
│     - Async vs Sync                         │
│     - Microservices vs Monolith             │
│                                             │
│  4. Ressources MongoDB                      │
│     - Connexions max par serveur            │
│     - RAM disponible                        │
│     - Nombre de replicas                    │
│                                             │
│  5. Contraintes réseau                      │
│     - Latence réseau                        │
│     - Bande passante                        │
│     - Firewalls                             │
└─────────────────────────────────────────────┘
```

### Recommandations par type d'application

#### Application Web Standard

```javascript
// Web app : 100-200 req/s, 20-50ms query time
const config = {
  maxPoolSize: 50,      // Suffisant pour la charge
  minPoolSize: 10,      // Warm pool pour répondre vite
  maxIdleTimeMS: 60000, // Nettoyer les connexions inactives
  waitQueueTimeoutMS: 3000  // Fail fast si surchargé
};

// Calcul :
// 200 req/s × 0.05s = 10 connexions théoriques
// × 2-5 pour marge = 20-50 connexions
```

#### API haute fréquence

```javascript
// API : 1000+ req/s, 10-30ms query time
const config = {
  maxPoolSize: 150,     // Haute capacité
  minPoolSize: 50,      // Pool toujours chaud
  maxIdleTimeMS: 120000,
  waitQueueTimeoutMS: 5000
};

// Calcul :
// 1000 req/s × 0.03s = 30 connexions théoriques
// × 4-5 pour pics = 120-150 connexions
```

#### Workers Background

```javascript
// Workers : 10-20 jobs parallèles, 1-5s par job
const config = {
  maxPoolSize: 30,      // Jobs parallèles + marge
  minPoolSize: 5,       // Économie de ressources
  maxIdleTimeMS: 300000, // Connexions longue durée ok
  waitQueueTimeoutMS: 10000
};

// Calcul :
// 20 jobs × 3s = 60 / 60 = 1 connexion/job
// × 1.5 pour sécurité = 30 connexions
```

#### Microservices

```javascript
// Microservice : variable, isolé
const config = {
  maxPoolSize: 20,      // Limité par instance
  minPoolSize: 3,       // Minimal
  maxIdleTimeMS: 60000,
  waitQueueTimeoutMS: 3000
};

// Logique :
// Chaque instance indépendante
// Scaling horizontal plutôt que pool large
```

#### Serverless / Lambda

```javascript
// Lambda : cold starts fréquents
const config = {
  maxPoolSize: 5,       // Très limité
  minPoolSize: 1,       // 1 connexion minimum
  maxIdleTimeMS: 10000, // Courte durée
  waitQueueTimeoutMS: 2000
};

// Context :
// 1 instance Lambda = 1 container
// Pool partagé non optimal
// Privilégier Atlas Data API ou HTTP
```

#### Analytics / Reporting

```javascript
// Analytics : requêtes longues, moins fréquentes
const config = {
  maxPoolSize: 20,
  minPoolSize: 5,
  maxIdleTimeMS: 600000, // 10 minutes
  waitQueueTimeoutMS: 30000, // Tolérance haute
  socketTimeoutMS: 300000    // 5 minutes pour aggregations
};
```

## Monitoring du pool

### Métriques clés

```javascript
// Node.js - Collecter les métriques
class PoolMetrics {
  constructor() {
    this.metrics = {
      totalCreated: 0,
      totalClosed: 0,
      currentActive: 0,
      currentAvailable: 0,
      checkoutDuration: [],
      waitQueueSize: 0,
      timeouts: 0
    };
  }

  onConnectionCreated(event) {
    this.metrics.totalCreated++;
    this.metrics.currentAvailable++;
  }

  onConnectionClosed(event) {
    this.metrics.totalClosed++;
    this.metrics.currentAvailable--;
  }

  onConnectionCheckedOut(event) {
    this.metrics.currentActive++;
    this.metrics.currentAvailable--;
    event.checkoutTime = Date.now();
  }

  onConnectionCheckedIn(event) {
    this.metrics.currentActive--;
    this.metrics.currentAvailable++;

    if (event.checkoutTime) {
      const duration = Date.now() - event.checkoutTime;
      this.metrics.checkoutDuration.push(duration);

      // Garder seulement les 1000 dernières
      if (this.metrics.checkoutDuration.length > 1000) {
        this.metrics.checkoutDuration.shift();
      }
    }
  }

  getStats() {
    const durations = this.metrics.checkoutDuration;
    return {
      ...this.metrics,
      avgCheckoutDuration: durations.length > 0
        ? durations.reduce((a, b) => a + b, 0) / durations.length
        : 0,
      p95CheckoutDuration: durations.length > 0
        ? this.percentile(durations, 0.95)
        : 0,
      utilization: this.metrics.currentActive /
        (this.metrics.currentActive + this.metrics.currentAvailable)
    };
  }

  percentile(arr, p) {
    const sorted = [...arr].sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * p) - 1;
    return sorted[index];
  }
}

// Utilisation
const metrics = new PoolMetrics();

client.on('connectionCreated', (e) => metrics.onConnectionCreated(e));
client.on('connectionClosed', (e) => metrics.onConnectionClosed(e));
client.on('connectionCheckedOut', (e) => metrics.onConnectionCheckedOut(e));
client.on('connectionCheckedIn', (e) => metrics.onConnectionCheckedIn(e));

// Exposer les métriques (ex: endpoint Prometheus)
app.get('/metrics', (req, res) => {
  const stats = metrics.getStats();
  res.send(`
# HELP mongodb_pool_connections_active Active connections
# TYPE mongodb_pool_connections_active gauge
mongodb_pool_connections_active ${stats.currentActive}

# HELP mongodb_pool_connections_available Available connections
# TYPE mongodb_pool_connections_available gauge
mongodb_pool_connections_available ${stats.currentAvailable}

# HELP mongodb_pool_connections_created_total Total created connections
# TYPE mongodb_pool_connections_created_total counter
mongodb_pool_connections_created_total ${stats.totalCreated}

# HELP mongodb_pool_checkout_duration_avg Average checkout duration (ms)
# TYPE mongodb_pool_checkout_duration_avg gauge
mongodb_pool_checkout_duration_avg ${stats.avgCheckoutDuration}

# HELP mongodb_pool_utilization Pool utilization ratio
# TYPE mongodb_pool_utilization gauge
mongodb_pool_utilization ${stats.utilization}
  `);
});
```

### Dashboard de monitoring

```javascript
// Exemple avec monitoring en temps réel
class PoolDashboard {
  constructor(client) {
    this.stats = {
      active: 0,
      available: 0,
      created: 0,
      closed: 0,
      errors: 0,
      history: []
    };

    this.setupMonitoring(client);
    this.startReporting();
  }

  setupMonitoring(client) {
    client.on('connectionCreated', () => {
      this.stats.created++;
      this.stats.available++;
    });

    client.on('connectionClosed', () => {
      this.stats.closed++;
    });

    client.on('connectionCheckedOut', () => {
      this.stats.active++;
      this.stats.available--;
    });

    client.on('connectionCheckedIn', () => {
      this.stats.active--;
      this.stats.available++;
    });

    client.on('error', () => {
      this.stats.errors++;
    });
  }

  startReporting() {
    setInterval(() => {
      const snapshot = {
        timestamp: Date.now(),
        active: this.stats.active,
        available: this.stats.available,
        total: this.stats.active + this.stats.available,
        utilization: this.stats.active /
          (this.stats.active + this.stats.available || 1)
      };

      this.stats.history.push(snapshot);

      // Garder seulement 1 heure d'historique (à 1s interval)
      if (this.stats.history.length > 3600) {
        this.stats.history.shift();
      }

      this.logStats(snapshot);
    }, 1000);
  }

  logStats(snapshot) {
    console.log(`
Pool Stats:
  Active: ${snapshot.active}
  Available: ${snapshot.available}
  Total: ${snapshot.total}
  Utilization: ${(snapshot.utilization * 100).toFixed(1)}%
  Created: ${this.stats.created}
  Closed: ${this.stats.closed}
  Errors: ${this.stats.errors}
    `);
  }

  getHistory() {
    return this.stats.history;
  }
}
```

## Patterns et anti-patterns

### ✅ Patterns recommandés

#### 1. Singleton Client

```javascript
// ✅ CORRECT : Un client global pour l'application
// database.js
let client = null;

export async function getClient() {
  if (!client) {
    client = new MongoClient(uri, {
      maxPoolSize: 100,
      minPoolSize: 10
    });
    await client.connect();
  }
  return client;
}

// Utilisation dans les routes
app.get('/users', async (req, res) => {
  const client = await getClient();
  const users = await client.db().collection('users').find().toArray();
  res.json(users);
  // Pas de close() !
});
```

#### 2. Réutilisation des connexions

```javascript
// ✅ CORRECT : Emprunter et retourner
async function getUserById(id) {
  const client = await getClient();
  // La connexion est automatiquement empruntée
  const user = await client.db().collection('users').findOne({ _id: id });
  // Et automatiquement retournée
  return user;
}
```

#### 3. Gestion des timeouts

```javascript
// ✅ CORRECT : Timeout côté application
async function findWithTimeout(collection, query, timeoutMs = 5000) {
  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Query timeout')), timeoutMs)
  );

  const queryPromise = collection.find(query).toArray();

  return Promise.race([queryPromise, timeoutPromise]);
}
```

#### 4. Monitoring proactif

```javascript
// ✅ CORRECT : Alertes sur problèmes
class PoolHealthCheck {
  constructor(metrics) {
    this.metrics = metrics;
    this.thresholds = {
      highUtilization: 0.8,  // 80%
      lowAvailable: 5,
      highWaitQueue: 10
    };
  }

  check() {
    const stats = this.metrics.getStats();
    const alerts = [];

    if (stats.utilization > this.thresholds.highUtilization) {
      alerts.push({
        level: 'warning',
        message: `High pool utilization: ${(stats.utilization * 100).toFixed(1)}%`,
        action: 'Consider increasing maxPoolSize'
      });
    }

    if (stats.currentAvailable < this.thresholds.lowAvailable) {
      alerts.push({
        level: 'critical',
        message: `Low available connections: ${stats.currentAvailable}`,
        action: 'Increase maxPoolSize or investigate slow queries'
      });
    }

    return alerts;
  }
}
```

### ❌ Anti-patterns à éviter

#### 1. Nouveau client par requête

```javascript
// ❌ INCORRECT : Crée un nouveau pool à chaque fois !
app.get('/users', async (req, res) => {
  const client = new MongoClient(uri);  // ❌ MAUVAIS
  await client.connect();
  const users = await client.db().collection('users').find().toArray();
  await client.close();  // ❌ Détruit le pool
  res.json(users);
});

// ✅ CORRECT : Réutiliser le client
const client = new MongoClient(uri);
await client.connect();

app.get('/users', async (req, res) => {
  const users = await client.db().collection('users').find().toArray();
  res.json(users);
});
```

#### 2. Fermer les connexions manuellement

```javascript
// ❌ INCORRECT : Ne jamais close() dans le code métier
async function getUser(id) {
  const client = await getClient();
  const user = await client.db().collection('users').findOne({ _id: id });
  await client.close();  // ❌ Détruit tout le pool !
  return user;
}

// ✅ CORRECT : Laisser le pool gérer
async function getUser(id) {
  const client = await getClient();
  const user = await client.db().collection('users').findOne({ _id: id });
  return user;  // Connexion automatiquement retournée
}
```

#### 3. Pool trop grand

```javascript
// ❌ INCORRECT : Pool démesuré
const client = new MongoClient(uri, {
  maxPoolSize: 1000  // ❌ Trop grand !
});

// Problèmes :
// - Consomme trop de RAM
// - Surcharge MongoDB
// - Diminue les performances

// ✅ CORRECT : Dimensionnement approprié
const client = new MongoClient(uri, {
  maxPoolSize: 100  // ✅ Raisonnable pour web app
});
```

#### 4. Pas de timeout

```javascript
// ❌ INCORRECT : Attente infinie
const client = new MongoClient(uri, {
  maxPoolSize: 50,
  waitQueueTimeoutMS: 0  // ❌ Attend indéfiniment
});

// ✅ CORRECT : Timeout configuré
const client = new MongoClient(uri, {
  maxPoolSize: 50,
  waitQueueTimeoutMS: 5000  // ✅ Fail après 5s
});
```

#### 5. Ignorer les erreurs de pool

```javascript
// ❌ INCORRECT : Pas de gestion d'erreurs
client.on('connectionPoolCleared', () => {
  // Rien... ❌
});

// ✅ CORRECT : Réagir aux problèmes
client.on('connectionPoolCleared', (event) => {
  logger.error('Pool cleared, possible network issue', {
    serverId: event.serverId
  });

  // Alerter l'équipe ops
  alerting.notify('MongoDB pool cleared', event);

  // Potentiellement reconnecter
  reconnectWithBackoff();
});
```

## Troubleshooting

### Problème : Pool exhausted

```
Symptôme :
MongoWaitQueueFullError: Too many operations waiting for connection
MongoServerSelectionError: Server selection timed out
```

**Diagnostic** :

```javascript
// Vérifier l'utilisation du pool
function diagnosePoolExhaustion(metrics) {
  const stats = metrics.getStats();

  console.log('Pool Diagnostic:');
  console.log(`  Current active: ${stats.currentActive}`);
  console.log(`  Current available: ${stats.currentAvailable}`);
  console.log(`  Utilization: ${(stats.utilization * 100).toFixed(1)}%`);
  console.log(`  Avg checkout duration: ${stats.avgCheckoutDuration}ms`);
  console.log(`  P95 checkout duration: ${stats.p95CheckoutDuration}ms`);

  // Analyses
  if (stats.utilization > 0.9) {
    console.log('⚠️  Pool near capacity, increase maxPoolSize');
  }

  if (stats.avgCheckoutDuration > 1000) {
    console.log('⚠️  Slow queries detected, optimize or add indexes');
  }

  if (stats.currentAvailable === 0) {
    console.log('❌ No available connections, immediate action required');
  }
}
```

**Solutions** :

```javascript
// 1. Augmenter maxPoolSize
const client = new MongoClient(uri, {
  maxPoolSize: 200  // Augmenté de 100
});

// 2. Ajouter un timeout
const client = new MongoClient(uri, {
  waitQueueTimeoutMS: 5000  // Fail fast
});

// 3. Optimiser les requêtes lentes
// Identifier avec MongoDB Profiler ou logs
db.setProfilingLevel(1, { slowms: 100 });

// 4. Implémenter un circuit breaker
class CircuitBreaker {
  constructor(threshold = 5) {
    this.failures = 0;
    this.threshold = threshold;
    this.isOpen = false;
  }

  async execute(fn) {
    if (this.isOpen) {
      throw new Error('Circuit breaker is open');
    }

    try {
      const result = await fn();
      this.failures = 0;
      return result;
    } catch (error) {
      this.failures++;
      if (this.failures >= this.threshold) {
        this.isOpen = true;
        setTimeout(() => {
          this.isOpen = false;
          this.failures = 0;
        }, 60000);  // Reset après 1 minute
      }
      throw error;
    }
  }
}
```

### Problème : Connexions qui fuient (connection leak)

```
Symptôme :
Nombre de connexions augmente continuellement
Pool ne se remplit jamais d'available connections
```

**Diagnostic** :

```javascript
// Tracker les checkout sans checkin
class LeakDetector {
  constructor() {
    this.checkedOut = new Map();
  }

  onCheckOut(connectionId) {
    this.checkedOut.set(connectionId, {
      time: Date.now(),
      stack: new Error().stack
    });
  }

  onCheckIn(connectionId) {
    this.checkedOut.delete(connectionId);
  }

  findLeaks(timeoutMs = 10000) {
    const now = Date.now();
    const leaks = [];

    for (const [id, info] of this.checkedOut) {
      if (now - info.time > timeoutMs) {
        leaks.push({
          connectionId: id,
          duration: now - info.time,
          stack: info.stack
        });
      }
    }

    return leaks;
  }
}
```

**Solutions** :

```javascript
// 1. Toujours utiliser try/finally avec cursors
async function safeCursorUsage() {
  const client = await getClient();
  const cursor = client.db().collection('users').find({});

  try {
    const users = await cursor.toArray();
    return users;
  } finally {
    await cursor.close();  // ✅ Toujours close
  }
}

// 2. Utiliser maxIdleTimeMS pour cleanup
const client = new MongoClient(uri, {
  maxIdleTimeMS: 60000  // Ferme les connexions idle
});

// 3. Monitoring automatique
setInterval(() => {
  const leaks = leakDetector.findLeaks(10000);
  if (leaks.length > 0) {
    logger.error('Connection leaks detected', { leaks });
  }
}, 60000);
```

### Problème : Connexions qui se ferment prématurément

```
Symptôme :
MongoNetworkError: connection closed
Connexions se ferment de manière inattendue
```

**Solutions** :

```javascript
// 1. Vérifier maxIdleTimeMS
const client = new MongoClient(uri, {
  maxIdleTimeMS: 300000  // 5 minutes au lieu de 60s
});

// 2. Implémenter keepAlive
const client = new MongoClient(uri, {
  socketTimeoutMS: 0,  // Désactiver timeout socket
  keepAlive: true,
  keepAliveInitialDelay: 30000  // 30s
});

// 3. Gérer les reconnexions
client.on('serverHeartbeatFailed', async (event) => {
  logger.warn('Heartbeat failed', event);
  // Le driver reconnecte automatiquement
});

client.on('serverHeartbeatSucceeded', (event) => {
  logger.info('Heartbeat succeeded', event);
});
```

## Bonnes pratiques

### ✅ DO - À faire

```javascript
// 1. Un seul MongoClient par application
const client = new MongoClient(uri);
await client.connect();

// 2. Dimensionner selon la charge
const maxPoolSize = Math.max(
  expectedConcurrentOps,
  Math.ceil(peakRequestsPerSec * avgQueryTimeSec)
);

// 3. Configurer des timeouts
const client = new MongoClient(uri, {
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000,
  serverSelectionTimeoutMS: 5000,
  waitQueueTimeoutMS: 5000
});

// 4. Monitor le pool
client.on('connectionPoolCreated', logEvent);
client.on('connectionCreated', logEvent);
client.on('connectionClosed', logEvent);

// 5. Gérer les erreurs de pool
client.on('connectionPoolCleared', handlePoolCleared);

// 6. Tester la configuration
async function testPoolConfiguration() {
  const concurrent = 50;
  const promises = Array(concurrent).fill().map(() =>
    client.db().collection('test').findOne({})
  );

  const start = Date.now();
  await Promise.all(promises);
  const duration = Date.now() - start;

  console.log(`${concurrent} ops in ${duration}ms`);
  // Devrait être < 1000ms
}

// 7. Utiliser minPoolSize pour warm pool
const client = new MongoClient(uri, {
  maxPoolSize: 100,
  minPoolSize: 10  // 10% toujours prêt
});

// 8. Cleanup à l'arrêt
process.on('SIGTERM', async () => {
  await client.close();
  process.exit(0);
});

// 9. Externaliser la config
const config = {
  maxPoolSize: process.env.MONGO_MAX_POOL_SIZE || 100,
  minPoolSize: process.env.MONGO_MIN_POOL_SIZE || 10
};

// 10. Logger les métriques
setInterval(() => {
  const stats = metrics.getStats();
  logger.info('Pool metrics', stats);
}, 60000);
```

### ❌ DON'T - À éviter

```javascript
// ❌ 1. Nouveau client par opération
async function getUser(id) {
  const client = new MongoClient(uri);  // NON !
  // ...
}

// ❌ 2. Fermer dans le code métier
await client.close();  // NON, sauf shutdown app

// ❌ 3. Pool trop grand
maxPoolSize: 10000  // NON !

// ❌ 4. Pas de timeout
waitQueueTimeoutMS: 0  // NON !

// ❌ 5. Ignorer les erreurs
// (pas de handlers)

// ❌ 6. Pas de monitoring
// (pas de metrics)

// ❌ 7. Configuration hardcodée
maxPoolSize: 100  // À éviter, préférer config

// ❌ 8. Pas de cleanup
// (pas de SIGTERM handler)

// ❌ 9. Cursors non fermés
const cursor = collection.find({});
// ...
// Pas de cursor.close()  // NON !

// ❌ 10. Pas de tests de charge
// Toujours tester la config en charge
```

## Conclusion

Le connection pooling est essentiel pour les performances MongoDB. Les points clés :

1. **Un client unique** : Singleton pour toute l'application
2. **Dimensionnement approprié** : Selon charge et architecture
3. **Timeouts configurés** : Pour éviter les blocages
4. **Monitoring actif** : Métriques et alertes
5. **Pas de fermeture manuelle** : Laisser le pool gérer

**Configuration type production** :
- maxPoolSize: 50-150
- minPoolSize: 10-20
- maxIdleTimeMS: 60000
- waitQueueTimeoutMS: 5000
- Monitoring et alertes actifs

---


⏭️ [Gestion des erreurs et retry](/15-drivers-integration-applicative/11-gestion-erreurs-retry.md)
