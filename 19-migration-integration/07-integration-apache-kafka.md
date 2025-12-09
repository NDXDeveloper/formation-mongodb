🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.7 Intégration avec Apache Kafka

## Introduction

Apache Kafka est devenu l'épine dorsale des architectures event-driven modernes. Son intégration avec MongoDB permet de construire des pipelines de données temps réel, d'implémenter des patterns Event Sourcing et CQRS, et de synchroniser données entre microservices distribués. Cette section explore les architectures, patterns et configurations de production pour intégrer MongoDB et Kafka à l'échelle enterprise.

---

## 🎯 Architectures d'Intégration

### Vue d'ensemble des patterns

**1. MongoDB → Kafka (Change Data Capture)**
```
[MongoDB] ──Change Streams──▶ [Kafka] ──▶ [Consumers]
```
Use cases : Event streaming, audit logs, replication cross-datacenter

**2. Kafka → MongoDB (Event Sink)**
```
[Producers] ──▶ [Kafka] ──▶ [MongoDB]
```
Use cases : Event storage, analytics, materialized views

**3. Bidirectional (Event-Driven Architecture)**
```
[MongoDB] ◀──▶ [Kafka] ◀──▶ [Services]
```
Use cases : CQRS, Event Sourcing, microservices orchestration

### Architecture de Production Complète

```
┌─────────────────────────────────────────────────────────────────┐
│           Production Event-Driven Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              MongoDB Atlas Cluster                     │     │
│  │  Primary + 2 Secondaries (Multi-Region)                │     │
│  │                                                        │     │
│  │  Collections:                                          │     │
│  │  • orders (source of truth)                            │     │
│  │  • order_events (event sourcing)                       │     │
│  │  • read_models (CQRS projections)                      │     │
│  └────────────┬───────────────────────────────────────────┘     │
│               │                                                 │
│               │ Change Streams                                  │
│               ↓                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │         Kafka Connect Cluster (3 nodes)               │      │
│  │                                                       │      │
│  │  ┌────────────────┐  ┌────────────────┐               │      │
│  │  │ MongoDB Source │  │ MongoDB Sink   │               │      │
│  │  │   Connector    │  │   Connector    │               │      │
│  │  └───────┬────────┘  └───────┬────────┘               │      │
│  └──────────┼───────────────────┼────────────────────────┘      │
│             │                   │                               │
│             ↓                   ↑                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Apache Kafka Cluster                       │    │
│  │  9 brokers (3 per AZ), RF=3, min.insync.replicas=2      │    │
│  │                                                         │    │
│  │  Topics:                                                │    │
│  │  • mongodb.orders.changes (24 partitions)               │    │
│  │  • order.created                                        │    │
│  │  • order.updated                                        │    │
│  │  • order.completed                                      │    │
│  │  • inventory.reserved                                   │    │
│  │                                                         │    │
│  │  Retention: 7 days (168h)                               │    │
│  │  Compression: lz4                                       │    │
│  └──────────┬───────────────────────────────────────────┬──┘    │
│             │                                           │       │
│             ↓                                           ↓       │
│  ┌──────────────────┐                      ┌──────────────────┐ │
│  │ Consumer Group 1 │                      │ Consumer Group 2 │ │
│  │  • Analytics     │                      │  • Notifications │ │
│  │  • Warehouse     │                      │  • Audit Log     │ │
│  └──────────────────┘                      └──────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │           Schema Registry (3 nodes)                    │     │
│  │  • Avro schemas                                        │     │
│  │  • Schema evolution compatibility                      │     │
│  │  • Version management                                  │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │         Monitoring & Observability                     │     │
│  │  • Confluent Control Center                            │     │
│  │  • Prometheus + Grafana                                │     │
│  │  • Kafka lag monitoring                                │     │
│  │  • MongoDB Atlas metrics                               │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📤 MongoDB Source Connector (MongoDB → Kafka)

### Principe

Le **MongoDB Kafka Source Connector** capture les changements MongoDB via Change Streams et publie les événements dans Kafka. Idéal pour CDC (Change Data Capture) et event streaming.

### Configuration Production

**connector-mongodb-source.json**

```json
{
  "name": "mongodb-source-orders",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSourceConnector",
    "tasks.max": "3",

    "connection.uri": "mongodb+srv://kafka-user:${file:/secrets/mongodb-password}@cluster.mongodb.net/production?retryWrites=true&w=majority",

    "database": "production",
    "collection": "orders",

    "topic.prefix": "mongodb",
    "topic.separator": ".",
    "topic.suffix": ".changes",

    "pipeline": "[{\"$match\":{\"operationType\":{\"$in\":[\"insert\",\"update\",\"delete\"]}}},{\"$project\":{\"_id\":1,\"operationType\":1,\"fullDocument\":1,\"ns\":1,\"documentKey\":1}}]",

    "change.stream.full.document": "updateLookup",
    "change.stream.full.document.before.change": "whenAvailable",

    "publish.full.document.only": "false",
    "publish.full.document.only.tombstone.on.delete": "true",

    "copy.existing": "false",
    "copy.existing.max.threads": "4",
    "copy.existing.queue.size": "16000",

    "poll.max.batch.size": "1000",
    "poll.await.time.ms": "5000",

    "heartbeat.interval.ms": "10000",
    "heartbeat.topic.name": "mongodb.heartbeats",

    "output.format.key": "json",
    "output.format.value": "json",
    "output.json.formatter": "com.mongodb.kafka.connect.source.json.formatter.SimplifiedJson",

    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true",
    "errors.deadletterqueue.topic.name": "mongodb.dlq.orders",
    "errors.deadletterqueue.context.headers.enable": "true",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",

    "transforms": "unwrap,extractKey,addMetadata",

    "transforms.unwrap.type": "com.mongodb.kafka.connect.source.MongoTimestampConverter",

    "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.extractKey.field": "_id",

    "transforms.addMetadata.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addMetadata.timestamp.field": "kafka_timestamp",
    "transforms.addMetadata.static.field": "source",
    "transforms.addMetadata.static.value": "mongodb-production"
  }
}
```

### Déploiement

```bash
# Deploy connector via REST API
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d @connector-mongodb-source.json

# Check status
curl http://kafka-connect:8083/connectors/mongodb-source-orders/status | jq

# Pause connector
curl -X PUT http://kafka-connect:8083/connectors/mongodb-source-orders/pause

# Resume connector
curl -X PUT http://kafka-connect:8083/connectors/mongodb-source-orders/resume

# Delete connector
curl -X DELETE http://kafka-connect:8083/connectors/mongodb-source-orders
```

### Format des messages Kafka

**Message Key** (JSON)
```json
{
  "_id": "507f1f77bcf86cd799439011"
}
```

**Message Value - Insert** (JSON)
```json
{
  "_id": {
    "$oid": "507f1f77bcf86cd799439011"
  },
  "operationType": "insert",
  "fullDocument": {
    "_id": {
      "$oid": "507f1f77bcf86cd799439011"
    },
    "order_number": "ORD-2024-00123",
    "customer_id": "CUST-456",
    "order_date": {
      "$date": "2024-01-15T10:30:00.000Z"
    },
    "status": "pending",
    "items": [
      {
        "product_id": "PROD-789",
        "quantity": 2,
        "price": 29.99
      }
    ],
    "total": 59.98
  },
  "ns": {
    "db": "production",
    "coll": "orders"
  },
  "documentKey": {
    "_id": {
      "$oid": "507f1f77bcf86cd799439011"
    }
  },
  "clusterTime": {
    "$timestamp": {
      "t": 1705315800,
      "i": 1
    }
  },
  "source": "mongodb-production",
  "kafka_timestamp": 1705315800123
}
```

**Message Value - Update**
```json
{
  "_id": {
    "$oid": "507f1f77bcf86cd799439011"
  },
  "operationType": "update",
  "fullDocument": {
    "_id": {
      "$oid": "507f1f77bcf86cd799439011"
    },
    "order_number": "ORD-2024-00123",
    "status": "completed",
    "completed_at": {
      "$date": "2024-01-15T14:30:00.000Z"
    }
  },
  "updateDescription": {
    "updatedFields": {
      "status": "completed",
      "completed_at": {
        "$date": "2024-01-15T14:30:00.000Z"
      }
    },
    "removedFields": [],
    "truncatedArrays": []
  },
  "ns": {
    "db": "production",
    "coll": "orders"
  }
}
```

**Message Value - Delete**
```json
{
  "_id": {
    "$oid": "507f1f77bcf86cd799439011"
  },
  "operationType": "delete",
  "ns": {
    "db": "production",
    "coll": "orders"
  },
  "documentKey": {
    "_id": {
      "$oid": "507f1f77bcf86cd799439011"
    }
  },
  "fullDocument": null
}
```

---

## 📥 MongoDB Sink Connector (Kafka → MongoDB)

### Principe

Le **MongoDB Kafka Sink Connector** consomme messages Kafka et écrit dans MongoDB. Supporte upsert, delete, bulk writes et transformations.

### Configuration Production

**connector-mongodb-sink.json**

```json
{
  "name": "mongodb-sink-events",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
    "tasks.max": "6",

    "connection.uri": "mongodb+srv://kafka-user:${file:/secrets/mongodb-password}@cluster.mongodb.net/events?retryWrites=true&w=majority",

    "database": "events",
    "collection": "order_events",

    "topics": "order.created,order.updated,order.completed",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",

    "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneDefaultStrategy",

    "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInKeyStrategy",

    "delete.on.null.values": "true",
    "delete.writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.DeleteOneDefaultStrategy",

    "max.num.retries": "3",
    "retries.defer.timeout": "5000",

    "bulk.write.ordered": "false",

    "rate.limiting.timeout": "0",
    "rate.limiting.every.n": "0",

    "change.data.capture.handler": "com.mongodb.kafka.connect.sink.cdc.mongodb.ChangeStreamHandler",

    "post.processor.chain": "com.mongodb.kafka.connect.sink.processor.DocumentIdAdder,com.mongodb.kafka.connect.sink.processor.BlockListValueProjector",

    "value.projection.type": "blocklist",
    "value.projection.list": "_id",

    "field.renamer.mapping": "[{\"oldName\":\"kafka_timestamp\",\"newName\":\"ingested_at\"}]",
    "field.renamer.regexp": "[]",

    "timeseries.timefield": "timestamp",
    "timeseries.timefield.auto.convert": "true",
    "timeseries.timefield.auto.convert.date.format": "yyyy-MM-dd'T'HH:mm:ss.SSSZ",

    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.deadletterqueue.topic.name": "mongodb.sink.dlq",
    "errors.deadletterqueue.context.headers.enable": "true",

    "transforms": "addTimestamp,flattenStruct",

    "transforms.addTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addTimestamp.timestamp.field": "processed_at",

    "transforms.flattenStruct.type": "org.apache.kafka.connect.transforms.Flatten$Value",
    "transforms.flattenStruct.delimiter": "_"
  }
}
```

### Stratégies d'écriture

**1. ReplaceOneDefaultStrategy (Upsert)**

```json
{
  "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneDefaultStrategy"
}
```

Génère :
```javascript
db.collection.replaceOne(
  { _id: <key> },
  <document>,
  { upsert: true }
)
```

**2. UpdateOneTimestampsStrategy (Update avec timestamps)**

```json
{
  "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.UpdateOneTimestampsStrategy"
}
```

Génère :
```javascript
db.collection.updateOne(
  { _id: <key> },
  {
    $set: <document>,
    $currentDate: {
      _ts: true,
      _modified: true
    }
  },
  { upsert: true }
)
```

**3. Custom Strategy (Business Logic)**

```java
// CustomUpsertStrategy.java
package com.example.kafka.mongodb;

import com.mongodb.kafka.connect.sink.converter.SinkDocument;
import com.mongodb.kafka.connect.sink.writemodel.strategy.WriteModelStrategy;
import com.mongodb.client.model.WriteModel;
import com.mongodb.client.model.UpdateOneModel;
import com.mongodb.client.model.UpdateOptions;
import org.bson.Document;

public class CustomUpsertStrategy implements WriteModelStrategy {

    @Override
    public WriteModel<Document> createWriteModel(SinkDocument document) {
        Document keyDoc = document.getKeyDoc().orElseThrow(
            () -> new IllegalStateException("Key document must be present")
        );

        Document valueDoc = document.getValueDoc().orElseThrow(
            () -> new IllegalStateException("Value document must be present")
        );

        // Business logic: Add version, timestamps, etc.
        Document update = new Document("$set", valueDoc)
            .append("$inc", new Document("version", 1))
            .append("$currentDate", new Document("updated_at", true));

        // Si nouveau document, ajouter created_at
        update.append("$setOnInsert", new Document("created_at", new Date()));

        return new UpdateOneModel<>(
            keyDoc,
            update,
            new UpdateOptions().upsert(true)
        );
    }
}
```

Configuration :
```json
{
  "writemodel.strategy": "com.example.kafka.mongodb.CustomUpsertStrategy"
}
```

---

## 🔄 Change Streams → Kafka (Custom Producer)

### Implémentation Node.js

**change-stream-producer.ts**

```typescript
// change-stream-producer.ts
import { MongoClient, ChangeStream, ChangeStreamDocument } from 'mongodb';
import { Kafka, Producer, CompressionTypes } from 'kafkajs';

interface ChangeStreamConfig {
  mongoUri: string;
  database: string;
  collection: string;
  kafkaBrokers: string[];
  kafkaTopic: string;
  batchSize: number;
  batchTimeout: number;
}

class ChangeStreamProducer {
  private mongoClient: MongoClient;
  private kafka: Kafka;
  private producer: Producer;
  private config: ChangeStreamConfig;
  private changeStream: ChangeStream | null = null;
  private buffer: any[] = [];
  private flushTimer: NodeJS.Timeout | null = null;

  constructor(config: ChangeStreamConfig) {
    this.config = config;

    this.mongoClient = new MongoClient(config.mongoUri);

    this.kafka = new Kafka({
      clientId: 'mongodb-change-stream-producer',
      brokers: config.kafkaBrokers,
      ssl: true,
      sasl: {
        mechanism: 'scram-sha-256',
        username: process.env.KAFKA_USERNAME,
        password: process.env.KAFKA_PASSWORD
      },
      retry: {
        retries: 5,
        initialRetryTime: 300,
        multiplier: 2
      }
    });

    this.producer = this.kafka.producer({
      allowAutoTopicCreation: false,
      transactionTimeout: 60000,
      compression: CompressionTypes.LZ4,
      maxInFlightRequests: 5,
      idempotent: true,
      retry: {
        retries: 5
      }
    });
  }

  async start(): Promise<void> {
    await this.mongoClient.connect();
    await this.producer.connect();

    const db = this.mongoClient.db(this.config.database);
    const collection = db.collection(this.config.collection);

    // Pipeline pour filtrer/transformer events
    const pipeline = [
      {
        $match: {
          operationType: { $in: ['insert', 'update', 'delete', 'replace'] }
        }
      },
      {
        $project: {
          _id: 1,
          operationType: 1,
          fullDocument: 1,
          fullDocumentBeforeChange: 1,
          updateDescription: 1,
          ns: 1,
          documentKey: 1,
          clusterTime: 1
        }
      }
    ];

    // Options Change Stream
    const options = {
      fullDocument: 'updateLookup' as const,
      fullDocumentBeforeChange: 'whenAvailable' as const
    };

    this.changeStream = collection.watch(pipeline, options);

    console.log(`Change Stream started on ${this.config.database}.${this.config.collection}`);

    this.changeStream.on('change', async (change: ChangeStreamDocument) => {
      await this.handleChange(change);
    });

    this.changeStream.on('error', (error) => {
      console.error('Change Stream error:', error);
      // Implement retry logic
      this.restart();
    });

    this.changeStream.on('close', () => {
      console.log('Change Stream closed');
      this.restart();
    });

    // Setup flush timer
    this.setupFlushTimer();
  }

  private async handleChange(change: ChangeStreamDocument): Promise<void> {
    const event = this.transformChangeEvent(change);

    this.buffer.push({
      topic: this.config.kafkaTopic,
      key: this.extractKey(change),
      value: JSON.stringify(event),
      headers: {
        'operation': change.operationType,
        'source': 'mongodb-change-stream',
        'timestamp': Date.now().toString()
      }
    });

    // Flush if buffer full
    if (this.buffer.length >= this.config.batchSize) {
      await this.flush();
    }
  }

  private transformChangeEvent(change: ChangeStreamDocument): any {
    const base = {
      _id: change._id,
      operationType: change.operationType,
      ns: change.ns,
      documentKey: change.documentKey,
      clusterTime: change.clusterTime,
      timestamp: new Date().toISOString()
    };

    switch (change.operationType) {
      case 'insert':
      case 'replace':
        return {
          ...base,
          fullDocument: change.fullDocument
        };

      case 'update':
        return {
          ...base,
          fullDocument: change.fullDocument,
          fullDocumentBeforeChange: change.fullDocumentBeforeChange,
          updateDescription: change.updateDescription
        };

      case 'delete':
        return {
          ...base,
          fullDocumentBeforeChange: change.fullDocumentBeforeChange
        };

      default:
        return base;
    }
  }

  private extractKey(change: ChangeStreamDocument): string {
    if (change.documentKey && change.documentKey._id) {
      return change.documentKey._id.toString();
    }
    return change._id.toString();
  }

  private setupFlushTimer(): void {
    this.flushTimer = setInterval(async () => {
      if (this.buffer.length > 0) {
        await this.flush();
      }
    }, this.config.batchTimeout);
  }

  private async flush(): Promise<void> {
    if (this.buffer.length === 0) return;

    const messages = this.buffer.splice(0);

    try {
      await this.producer.sendBatch({
        topicMessages: [
          {
            topic: this.config.kafkaTopic,
            messages
          }
        ]
      });

      console.log(`Flushed ${messages.length} messages to Kafka`);

      // Metrics
      this.recordMetrics({
        messages_sent: messages.length,
        timestamp: new Date()
      });

    } catch (error) {
      console.error('Error flushing to Kafka:', error);

      // Re-add to buffer for retry
      this.buffer.unshift(...messages);

      // Exponential backoff
      await this.sleep(1000);
      await this.flush();
    }
  }

  private async restart(): Promise<void> {
    console.log('Restarting Change Stream...');

    if (this.changeStream) {
      await this.changeStream.close();
    }

    if (this.flushTimer) {
      clearInterval(this.flushTimer);
    }

    // Wait before restart
    await this.sleep(5000);

    await this.start();
  }

  private async sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  private recordMetrics(metrics: any): void {
    // Send to metrics system (Prometheus, DataDog, etc.)
  }

  async stop(): Promise<void> {
    console.log('Stopping Change Stream Producer...');

    if (this.flushTimer) {
      clearInterval(this.flushTimer);
    }

    // Flush remaining messages
    await this.flush();

    if (this.changeStream) {
      await this.changeStream.close();
    }

    await this.producer.disconnect();
    await this.mongoClient.close();

    console.log('Stopped');
  }
}

// Usage
const config: ChangeStreamConfig = {
  mongoUri: process.env.MONGODB_URI,
  database: 'production',
  collection: 'orders',
  kafkaBrokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
  kafkaTopic: 'mongodb.orders.changes',
  batchSize: 1000,
  batchTimeout: 5000  // 5 seconds
};

const producer = new ChangeStreamProducer(config);

process.on('SIGTERM', async () => {
  await producer.stop();
  process.exit(0);
});

producer.start().catch(console.error);
```

---

## 🎭 Event Sourcing avec MongoDB + Kafka

### Architecture Event Sourcing

```
┌─────────────────────────────────────────────────────────┐
│            Event Sourcing Architecture                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Command] ──▶ [Command Handler] ──▶ [Event Store]      │
│                       │                    │            │
│                       │                    │ (MongoDB)  │
│                       ↓                    ↓            │
│                [Business Logic]  [Append-Only Events]   │
│                       │                    │            │
│                       │                    ↓            │
│                       │           [Kafka (Stream)]      │
│                       │                    │            │
│                       │           ┌────────┴────────┐   │
│                       │           ↓                 ↓   │
│                       │    [Projections]     [Reactions]│
│                       │           │                 │   │
│                       ↓           ↓                 ↓   │
│                [Read Models]  [Notifications]  [Sagas]  │
│                    (MongoDB)                            │
└─────────────────────────────────────────────────────────┘
```

### Implémentation

**Event Store (MongoDB)**

```typescript
// event-store.ts
import { MongoClient, Db, Collection } from 'mongodb';
import { v4 as uuidv4 } from 'uuid';

interface DomainEvent {
  event_id: string;
  event_type: string;
  aggregate_type: string;
  aggregate_id: string;
  version: number;
  timestamp: Date;
  data: any;
  metadata?: any;
}

class EventStore {
  private db: Db;
  private events: Collection<DomainEvent>;

  constructor(mongoClient: MongoClient) {
    this.db = mongoClient.db('event_store');
    this.events = this.db.collection('events');
  }

  async appendEvent(event: Omit<DomainEvent, 'event_id' | 'timestamp'>): Promise<DomainEvent> {
    // Optimistic concurrency control
    const existingEvents = await this.events.countDocuments({
      aggregate_type: event.aggregate_type,
      aggregate_id: event.aggregate_id
    });

    if (existingEvents !== event.version - 1) {
      throw new Error('Concurrency conflict');
    }

    const fullEvent: DomainEvent = {
      event_id: uuidv4(),
      timestamp: new Date(),
      ...event
    };

    await this.events.insertOne(fullEvent);

    // Publish to Kafka
    await this.publishToKafka(fullEvent);

    return fullEvent;
  }

  async getEvents(
    aggregateType: string,
    aggregateId: string,
    fromVersion: number = 0
  ): Promise<DomainEvent[]> {
    return await this.events
      .find({
        aggregate_type: aggregateType,
        aggregate_id: aggregateId,
        version: { $gt: fromVersion }
      })
      .sort({ version: 1 })
      .toArray();
  }

  async getEventsByType(
    eventType: string,
    fromTimestamp?: Date
  ): Promise<DomainEvent[]> {
    const query: any = { event_type: eventType };

    if (fromTimestamp) {
      query.timestamp = { $gte: fromTimestamp };
    }

    return await this.events
      .find(query)
      .sort({ timestamp: 1 })
      .toArray();
  }

  private async publishToKafka(event: DomainEvent): Promise<void> {
    // Publish via Kafka producer (implementation omitted for brevity)
  }
}

// Aggregate example: Order
interface Order {
  id: string;
  customer_id: string;
  items: Array<{ product_id: string; quantity: number; price: number }>;
  status: 'pending' | 'confirmed' | 'shipped' | 'completed' | 'cancelled';
  total: number;
  version: number;
}

class OrderAggregate {
  private state: Order;
  private uncommittedEvents: DomainEvent[] = [];

  constructor(state: Order) {
    this.state = state;
  }

  // Command: Create Order
  createOrder(customerId: string, items: any[]): void {
    if (this.state.version !== 0) {
      throw new Error('Order already exists');
    }

    const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

    this.applyEvent({
      event_type: 'OrderCreated',
      aggregate_type: 'Order',
      aggregate_id: this.state.id,
      version: 1,
      data: {
        customer_id: customerId,
        items,
        total
      }
    });
  }

  // Command: Confirm Order
  confirmOrder(): void {
    if (this.state.status !== 'pending') {
      throw new Error('Order must be pending');
    }

    this.applyEvent({
      event_type: 'OrderConfirmed',
      aggregate_type: 'Order',
      aggregate_id: this.state.id,
      version: this.state.version + 1,
      data: {
        confirmed_at: new Date()
      }
    });
  }

  // Command: Ship Order
  shipOrder(trackingNumber: string): void {
    if (this.state.status !== 'confirmed') {
      throw new Error('Order must be confirmed');
    }

    this.applyEvent({
      event_type: 'OrderShipped',
      aggregate_type: 'Order',
      aggregate_id: this.state.id,
      version: this.state.version + 1,
      data: {
        tracking_number: trackingNumber,
        shipped_at: new Date()
      }
    });
  }

  private applyEvent(event: Omit<DomainEvent, 'event_id' | 'timestamp'>): void {
    // Apply event to state
    switch (event.event_type) {
      case 'OrderCreated':
        this.state.customer_id = event.data.customer_id;
        this.state.items = event.data.items;
        this.state.total = event.data.total;
        this.state.status = 'pending';
        this.state.version = event.version;
        break;

      case 'OrderConfirmed':
        this.state.status = 'confirmed';
        this.state.version = event.version;
        break;

      case 'OrderShipped':
        this.state.status = 'shipped';
        this.state.version = event.version;
        break;
    }

    this.uncommittedEvents.push(event as DomainEvent);
  }

  async commit(eventStore: EventStore): Promise<void> {
    for (const event of this.uncommittedEvents) {
      await eventStore.appendEvent(event);
    }

    this.uncommittedEvents = [];
  }

  static async load(
    eventStore: EventStore,
    orderId: string
  ): Promise<OrderAggregate> {
    const events = await eventStore.getEvents('Order', orderId);

    const order: Order = {
      id: orderId,
      customer_id: '',
      items: [],
      status: 'pending',
      total: 0,
      version: 0
    };

    const aggregate = new OrderAggregate(order);

    for (const event of events) {
      aggregate.applyEvent(event);
    }

    return aggregate;
  }
}

// Usage
const eventStore = new EventStore(mongoClient);

// Create order
const order = new OrderAggregate({
  id: uuidv4(),
  customer_id: '',
  items: [],
  status: 'pending',
  total: 0,
  version: 0
});

order.createOrder('CUST-123', [
  { product_id: 'PROD-1', quantity: 2, price: 29.99 },
  { product_id: 'PROD-2', quantity: 1, price: 49.99 }
]);

await order.commit(eventStore);

// Later: Load and modify
const existingOrder = await OrderAggregate.load(eventStore, order.id);
existingOrder.confirmOrder();
await existingOrder.commit(eventStore);
```

### Projection (Read Model)

```typescript
// order-projection.ts
import { Kafka, Consumer } from 'kafkajs';
import { MongoClient, Db, Collection } from 'mongodb';

interface OrderReadModel {
  _id: string;
  order_number: string;
  customer_id: string;
  customer_name: string;
  items: any[];
  status: string;
  total: number;
  created_at: Date;
  updated_at: Date;
}

class OrderProjection {
  private consumer: Consumer;
  private db: Db;
  private orders: Collection<OrderReadModel>;

  constructor(kafka: Kafka, mongoClient: MongoClient) {
    this.consumer = kafka.consumer({ groupId: 'order-projection' });
    this.db = mongoClient.db('read_models');
    this.orders = this.db.collection('orders');
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topics: ['order.created', 'order.confirmed', 'order.shipped'],
      fromBeginning: false
    });

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const event = JSON.parse(message.value.toString());

        await this.handleEvent(event);
      }
    });
  }

  private async handleEvent(event: DomainEvent): Promise<void> {
    switch (event.event_type) {
      case 'OrderCreated':
        await this.handleOrderCreated(event);
        break;

      case 'OrderConfirmed':
        await this.handleOrderConfirmed(event);
        break;

      case 'OrderShipped':
        await this.handleOrderShipped(event);
        break;
    }
  }

  private async handleOrderCreated(event: DomainEvent): Promise<void> {
    const readModel: OrderReadModel = {
      _id: event.aggregate_id,
      order_number: `ORD-${Date.now()}`,
      customer_id: event.data.customer_id,
      customer_name: await this.getCustomerName(event.data.customer_id),
      items: event.data.items,
      status: 'pending',
      total: event.data.total,
      created_at: event.timestamp,
      updated_at: event.timestamp
    };

    await this.orders.insertOne(readModel);
  }

  private async handleOrderConfirmed(event: DomainEvent): Promise<void> {
    await this.orders.updateOne(
      { _id: event.aggregate_id },
      {
        $set: {
          status: 'confirmed',
          confirmed_at: event.data.confirmed_at,
          updated_at: event.timestamp
        }
      }
    );
  }

  private async handleOrderShipped(event: DomainEvent): Promise<void> {
    await this.orders.updateOne(
      { _id: event.aggregate_id },
      {
        $set: {
          status: 'shipped',
          tracking_number: event.data.tracking_number,
          shipped_at: event.data.shipped_at,
          updated_at: event.timestamp
        }
      }
    );
  }

  private async getCustomerName(customerId: string): Promise<string> {
    // Fetch from customer service or read model
    return 'Customer Name';
  }
}
```

---

## 🔁 CQRS Pattern avec MongoDB + Kafka

### Architecture CQRS

```
┌─────────────────────────────────────────────────────────┐
│                   CQRS Architecture                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [Write Side]                      [Read Side]          │
│                                                         │
│  [Commands]                        [Queries]            │
│      ↓                                  ↑               │
│  [Command Bus]                     [Query Bus]          │
│      ↓                                  ↑               │
│  [Command Handlers]               [Query Handlers]      │
│      ↓                                  ↑               │
│  [Aggregates]                      [Read Models]        │
│      ↓                                  ↑               │
│  [Event Store]                     [Projection DB]      │
│   (MongoDB)                         (MongoDB)           │
│      │                                  ↑               │
│      └──────▶ [Kafka] ──────────────────┘               │
│             (Event Bus)                                 │
│                                                         │
│  Eventual Consistency: Events propagate from            │
│  write side to read side via Kafka                      │
└─────────────────────────────────────────────────────────┘
```

### Implémentation simplifiée

**Write Side (Commands)**

```typescript
// commands.ts
interface CreateOrderCommand {
  type: 'CreateOrder';
  order_id: string;
  customer_id: string;
  items: Array<{ product_id: string; quantity: number; price: number }>;
}

interface ConfirmOrderCommand {
  type: 'ConfirmOrder';
  order_id: string;
}

type OrderCommand = CreateOrderCommand | ConfirmOrderCommand;

class CommandBus {
  private handlers: Map<string, (cmd: any) => Promise<void>>;

  constructor() {
    this.handlers = new Map();
  }

  register(commandType: string, handler: (cmd: any) => Promise<void>): void {
    this.handlers.set(commandType, handler);
  }

  async dispatch(command: OrderCommand): Promise<void> {
    const handler = this.handlers.get(command.type);

    if (!handler) {
      throw new Error(`No handler for command ${command.type}`);
    }

    await handler(command);
  }
}

// Command Handlers
class OrderCommandHandlers {
  constructor(
    private eventStore: EventStore,
    private commandBus: CommandBus
  ) {
    commandBus.register('CreateOrder', this.handleCreateOrder.bind(this));
    commandBus.register('ConfirmOrder', this.handleConfirmOrder.bind(this));
  }

  private async handleCreateOrder(cmd: CreateOrderCommand): Promise<void> {
    const order = new OrderAggregate({
      id: cmd.order_id,
      customer_id: '',
      items: [],
      status: 'pending',
      total: 0,
      version: 0
    });

    order.createOrder(cmd.customer_id, cmd.items);

    await order.commit(this.eventStore);
  }

  private async handleConfirmOrder(cmd: ConfirmOrderCommand): Promise<void> {
    const order = await OrderAggregate.load(this.eventStore, cmd.order_id);

    order.confirmOrder();

    await order.commit(this.eventStore);
  }
}
```

**Read Side (Queries)**

```typescript
// queries.ts
interface GetOrderQuery {
  type: 'GetOrder';
  order_id: string;
}

interface SearchOrdersQuery {
  type: 'SearchOrders';
  customer_id?: string;
  status?: string;
  from_date?: Date;
  to_date?: Date;
}

type OrderQuery = GetOrderQuery | SearchOrdersQuery;

class QueryBus {
  private handlers: Map<string, (query: any) => Promise<any>>;

  constructor() {
    this.handlers = new Map();
  }

  register(queryType: string, handler: (query: any) => Promise<any>): void {
    this.handlers.set(queryType, handler);
  }

  async dispatch(query: OrderQuery): Promise<any> {
    const handler = this.handlers.get(query.type);

    if (!handler) {
      throw new Error(`No handler for query ${query.type}`);
    }

    return await handler(query);
  }
}

// Query Handlers
class OrderQueryHandlers {
  constructor(
    private ordersReadModel: Collection<OrderReadModel>,
    private queryBus: QueryBus
  ) {
    queryBus.register('GetOrder', this.handleGetOrder.bind(this));
    queryBus.register('SearchOrders', this.handleSearchOrders.bind(this));
  }

  private async handleGetOrder(query: GetOrderQuery): Promise<OrderReadModel> {
    const order = await this.ordersReadModel.findOne({ _id: query.order_id });

    if (!order) {
      throw new Error('Order not found');
    }

    return order;
  }

  private async handleSearchOrders(query: SearchOrdersQuery): Promise<OrderReadModel[]> {
    const filter: any = {};

    if (query.customer_id) {
      filter.customer_id = query.customer_id;
    }

    if (query.status) {
      filter.status = query.status;
    }

    if (query.from_date || query.to_date) {
      filter.created_at = {};
      if (query.from_date) filter.created_at.$gte = query.from_date;
      if (query.to_date) filter.created_at.$lte = query.to_date;
    }

    return await this.ordersReadModel
      .find(filter)
      .sort({ created_at: -1 })
      .toArray();
  }
}
```

---

## 📊 Scénario Réel : E-commerce Platform (12 mois)

**Contexte**
- E-commerce B2C, 10M utilisateurs
- MongoDB Atlas : 500 Go (orders, products, users)
- Kafka : 9 brokers, 300 topics, 50K msg/sec
- Services : 25 microservices (Node.js, Java, Go)
- Objectif : Architecture event-driven complète avec CDC, Event Sourcing, CQRS

### Architecture finale

```
┌──────────────────────────────────────────────────────────────┐
│               Production Event-Driven Platform               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Write Side (Command)                                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Order Service ──▶ MongoDB Event Store                 │  │
│  │                    (append-only events)                │  │
│  │                           │                            │  │
│  │                           ↓                            │  │
│  │                    Change Streams → Kafka              │  │
│  └────────────────────────────────────────────────────────┘  │
│                              │                               │
│                              ↓                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │          Kafka Cluster (Event Bus)                     │  │
│  │  • 9 brokers (3 AZs)                                   │  │
│  │  • Topics: order.*, inventory.*, payment.*             │  │
│  │  • 50K msg/sec throughput                              │  │
│  │  • 7 days retention                                    │  │
│  └──────────┬───────────────────────────────────────┬─────┘  │
│             │                                       │        │
│             ↓                                       ↓        │
│  Read Side (Query)                      Reactions/Sagas      │
│  ┌───────────────────────┐    ┌─────────────────────────┐    │
│  │  Projections          │    │  • Notification Service │    │
│  │  (MongoDB)            │    │  • Email Service        │    │
│  │  • orders_summary     │    │  • Inventory Service    │    │
│  │  • customer_360       │    │  • Payment Service      │    │
│  │  • analytics_daily    │    │  • Shipping Service     │    │
│  └───────────────────────┘    └─────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Métriques de production

**Kafka**
- Messages/sec : 50,000 (P95), 75,000 (peak)
- Latency end-to-end : 150ms (P95)
- Retention : 7 days (168h)
- Topics : 312 topics actifs
- Consumer lag : < 1000 messages (P95)

**MongoDB**
- Collections : 45 collections
- Event Store : 2 milliards d'événements (250 Go)
- Read Models : 450 Go
- Change Streams : 8 actifs
- Write throughput : 15K ops/sec

**Services**
- Order Service : 500 req/sec
- Projection Services : 6 instances (auto-scaling)
- Latency API : 50ms (P95)

### Timeline de déploiement

**Mois 1-3 : Infrastructure**
- Setup Kafka cluster (9 brokers)
- MongoDB Atlas M100 clusters
- Schema Registry
- Monitoring (Prometheus, Grafana, Datadog)

**Mois 4-6 : MVP Event Sourcing**
- Order Service avec Event Store
- Kafka Connect (Source connector)
- 3 projections basiques
- Tests charge

**Mois 7-9 : CQRS Complet**
- Command/Query separation
- 10 projections métier
- Sagas (orchestration distribuée)
- Optimisations performance

**Mois 10-12 : Migration Complète**
- 25 microservices migrés
- CDC sur toutes collections critiques
- Decommissioning ancien monolithe
- Production readiness

### Résultats

**Performance**
- Latency API : 250ms → 50ms
- Throughput : 5K → 15K req/sec
- Scalability : 100K → 10M users (horizontal scaling)

**Résilience**
- Availability : 99.9% → 99.99%
- MTTR : 2h → 15min
- Zero data loss sur 12 mois

**Business**
- Time-to-market : -60% (features indépendantes)
- Coûts infra : -25% (optimisation)
- Developer velocity : +80%

### Challenges

**Challenge 1 : Kafka lag spikes**
- **Symptôme** : Consumer lag monte à 500K messages
- **Cause** : Projection service saturé (single instance)
- **Solution** : Auto-scaling projections (6 instances), partitioning optimisé

**Challenge 2 : Duplicate events**
- **Symptôme** : 0.1% events dupliqués (réseau instable)
- **Cause** : Retry Kafka producer sans idempotence
- **Solution** : Idempotent producer + deduplication côté consumer

**Challenge 3 : Schema evolution**
- **Symptôme** : Breaking changes cassent consumers
- **Cause** : Pas de versioning events
- **Solution** : Schema Registry + backward compatibility enforcement

---

## 🎯 Bonnes Pratiques

### 1. Exactly-Once Semantics

```typescript
// Exactly-once processing avec Kafka Transactions
class ExactlyOnceProcessor {
  private consumer: Consumer;
  private producer: Producer;

  async processWithTransaction(): Promise<void> {
    await this.consumer.run({
      eachBatch: async ({
        batch,
        resolveOffset,
        heartbeat,
        isRunning,
        isStale
      }) => {
        const transaction = await this.producer.transaction();

        try {
          for (const message of batch.messages) {
            // Process message
            const event = JSON.parse(message.value.toString());

            // Write to MongoDB
            await this.processEvent(event);

            // Produce to another topic (within transaction)
            await transaction.send({
              topic: 'processed-events',
              messages: [{
                key: message.key,
                value: JSON.stringify({ ...event, processed: true })
              }]
            });

            // Commit offset (within transaction)
            await transaction.sendOffsets({
              consumerGroupId: 'my-consumer-group',
              topics: [{
                topic: batch.topic,
                partitions: [{
                  partition: batch.partition,
                  offset: message.offset
                }]
              }]
            });
          }

          await transaction.commit();

        } catch (error) {
          await transaction.abort();
          throw error;
        }
      }
    });
  }
}
```

### 2. Schema Registry Integration

```typescript
// Avro schema definition
const orderCreatedSchema = {
  type: 'record',
  name: 'OrderCreated',
  namespace: 'com.example.events',
  fields: [
    { name: 'order_id', type: 'string' },
    { name: 'customer_id', type: 'string' },
    { name: 'items', type: {
      type: 'array',
      items: {
        type: 'record',
        name: 'OrderItem',
        fields: [
          { name: 'product_id', type: 'string' },
          { name: 'quantity', type: 'int' },
          { name: 'price', type: 'double' }
        ]
      }
    }},
    { name: 'total', type: 'double' },
    { name: 'timestamp', type: 'long' }
  ]
};

// Producer with Schema Registry
import { SchemaRegistry } from '@kafkajs/confluent-schema-registry';

const registry = new SchemaRegistry({
  host: 'http://schema-registry:8081'
});

const producer = kafka.producer();

await producer.send({
  topic: 'order.created',
  messages: [{
    key: orderId,
    value: await registry.encode(schemaId, {
      order_id: orderId,
      customer_id: customerId,
      items: [...],
      total: 199.99,
      timestamp: Date.now()
    })
  }]
});
```

### 3. Monitoring

```typescript
// Comprehensive monitoring
interface KafkaMetrics {
  producer: {
    messages_sent_per_sec: number;
    error_rate: number;
    batch_size_avg: number;
    compression_rate: number;
  };
  consumer: {
    messages_consumed_per_sec: number;
    lag_messages: number;
    lag_time_ms: number;
    error_rate: number;
  };
  mongodb: {
    writes_per_sec: number;
    reads_per_sec: number;
    change_stream_lag_ms: number;
  };
}

// Alerting thresholds
const ALERT_THRESHOLDS = {
  kafka_lag: 10000,              // 10K messages
  kafka_lag_time: 60000,         // 1 minute
  mongodb_change_stream_lag: 5000,  // 5 seconds
  error_rate: 0.01               // 1%
};
```

---

## 📚 Checklist MongoDB + Kafka

**Infrastructure**
- [ ] Kafka cluster configuré (9+ brokers, RF=3)
- [ ] MongoDB Atlas clusters (write/read separation)
- [ ] Schema Registry déployé
- [ ] Monitoring stack (Prometheus, Grafana)
- [ ] Alerting configuré

**Connectivity**
- [ ] MongoDB Source Connector déployé
- [ ] MongoDB Sink Connector déployé
- [ ] Change Streams configurés
- [ ] Topics Kafka créés avec bon partitioning
- [ ] Dead Letter Queues configurées

**Performance**
- [ ] Indexes MongoDB optimisés
- [ ] Kafka partitions alignées sur use cases
- [ ] Consumer groups configurés (max.poll.records, etc.)
- [ ] Batching optimisé (producer et consumer)
- [ ] Compression activée (lz4 recommandé)

**Reliability**
- [ ] Idempotent producer activé
- [ ] Exactly-once semantics configuré (si nécessaire)
- [ ] Retry logic robuste
- [ ] Error handling et DLQ
- [ ] Circuit breakers

**Security**
- [ ] SSL/TLS activé (Kafka ↔ Clients)
- [ ] SASL authentication configurée
- [ ] MongoDB users dédiés avec permissions minimales
- [ ] Schema Registry sécurisé
- [ ] Network isolation

**Monitoring**
- [ ] Kafka lag monitoring
- [ ] Change Streams lag monitoring
- [ ] Error rate tracking
- [ ] Throughput metrics
- [ ] Latency P50/P95/P99

---

## 🎓 Conclusion

L'intégration MongoDB + Kafka est **essentielle** pour architectures event-driven modernes. Points clés :

**Use Cases**
- ✅ CDC (Change Data Capture) temps réel
- ✅ Event Sourcing et audit immutable
- ✅ CQRS (separation read/write)
- ✅ Microservices asynchronous communication
- ✅ Real-time analytics pipelines

**Patterns recommandés**
- MongoDB Source Connector pour CDC simple
- Change Streams custom pour contrôle fin
- Event Store MongoDB pour Event Sourcing
- Projections MongoDB pour CQRS read models

**Scalabilité**
- Kafka : 100K+ messages/sec
- MongoDB Change Streams : 10K+ ops/sec
- Latency end-to-end : < 200ms P95

L'intégration MongoDB + Kafka permet de construire des systèmes **résilients**, **scalables** et **event-driven** à l'échelle enterprise. Investissement initial important mais ROI significatif sur durabilité et évolutivité.

---

**Prochaine section** : 19.8 Intégration avec Data Lakes (S3, Delta Lake) - Patterns d'export MongoDB vers data lakes pour analytics at scale.

⏭️ [Intégration avec Apache Spark](/19-migration-integration/08-integration-apache-spark.md)
