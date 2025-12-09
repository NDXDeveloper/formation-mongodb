🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.10 Coexistence avec des Bases Relationnelles

## Introduction

Dans le monde réel, rares sont les organisations qui peuvent se permettre un "Big Bang" complet vers MongoDB. La coexistence entre bases relationnelles (PostgreSQL, MySQL, Oracle) et MongoDB est non seulement fréquente mais souvent souhaitable. Cette approche "Polyglot Persistence" permet d'exploiter les forces de chaque technologie : ACID pour transactions critiques (SQL) et flexibilité/scalabilité pour données semi-structurées (MongoDB).

Cette section explore les patterns architecturaux, stratégies de synchronisation et meilleures pratiques pour faire coexister harmonieusement SQL et MongoDB en production.

---

## 🎯 Polyglot Persistence

### Principe fondamental

**Polyglot Persistence** : Utiliser différentes technologies de stockage pour différents besoins métier au sein d'une même application.

```
┌────────────────────────────────────────────────────────────────┐
│              Polyglot Persistence Architecture                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [Application Layer]                                           │
│           │                                                    │
│           ├──────────────┬──────────────┬─────────────┐        │
│           ↓              ↓              ↓             ↓        │
│  ┌─────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ PostgreSQL  │  │ MongoDB  │  │  Redis   │  │   S3     │     │
│  │ (ACID txn)  │  │(flexible)│  │ (cache)  │  │ (archive)│     │
│  └─────────────┘  └──────────┘  └──────────┘  └──────────┘     │
│        │                │              │             │         │
│  • Orders         • Products     • Sessions   • Logs           │
│  • Invoices       • Catalog      • Metrics    • Backups        │
│  • Payments       • Reviews      • Leaderboard                 │
│                                                                │
│  Criteria de choix :                                           │
│  • ACID strict → PostgreSQL                                    │
│  • Schema flexible → MongoDB                                   │
│  • Performance reads → Redis                                   │
│  • Long-term storage → S3                                      │
└────────────────────────────────────────────────────────────────┘
```

### Matrice de décision

| Use Case | SQL (PostgreSQL) | MongoDB | Justification |
|----------|------------------|---------|---------------|
| **Orders & Invoices** | ✅ Primary | ❌ | ACID transactions critiques |
| **Product Catalog** | ❌ | ✅ Primary | Schema variable (attributs produits) |
| **Customer Profiles** | ✅ Primary | ✅ Cache | Transactions + lectures rapides |
| **User Reviews** | ❌ | ✅ Primary | Documents semi-structurés |
| **Inventory** | ✅ Primary | ❌ | Contraintes stock strictes |
| **Logs & Events** | ❌ | ✅ Primary | Volume massif, time-series |
| **Analytics** | ❌ | ✅ Primary | Aggregations complexes |
| **Financial Reports** | ✅ Primary | ❌ | Audit trail, compliance |

---

## 🏗️ Architectures Hybrides

### Architecture 1 : Segregation par Bounded Context

**Principe** : Chaque microservice choisit sa base de données selon ses besoins.

```
┌────────────────────────────────────────────────────────────────┐
│           Microservices avec Polyglot Persistence              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   API Gateway                            │  │
│  │  • Routing                                               │  │
│  │  • Authentication                                        │  │
│  │  • Rate limiting                                         │  │
│  └────────────┬───────────────────────────────────┬─────────┘  │
│               │                                   │            │
│       ┌───────┴────────┐                   ┌──────┴──────┐     │
│       ↓                ↓                   ↓             ↓     │
│  ┌─────────┐    ┌──────────┐       ┌──────────┐  ┌──────────┐  │
│  │ Order   │    │ Product  │       │  User    │  │Analytics │  │
│  │ Service │    │ Service  │       │ Service  │  │ Service  │  │
│  └────┬────┘    └────┬─────┘       └────┬─────┘  └────┬─────┘  │
│       │              │                  │             │        │
│       ↓              ↓                  ↓             ↓        │
│  ┌─────────┐    ┌──────────┐       ┌────────┐   ┌──────────┐   │
│  │PostgreSQL    │ MongoDB  │       │Postgres│   │ MongoDB  │   │
│  │         │    │          │       │  +     │   │          │   │
│  │• orders │    │• products│       │MongoDB │   │• events  │   │
│  │• invoices    │• catalog │       │(hybrid)│   │• metrics │   │
│  └─────────┘    └──────────┘       └────────┘   └──────────┘   │
│                                                                │
│  Benefits:                                                     │
│  • Independence des équipes                                    │
│  • Optimal database per use case                               │
│  • Scaling indépendant                                         │
└────────────────────────────────────────────────────────────────┘
```

### Architecture 2 : Dual Database avec Synchronisation

**Principe** : Mêmes données dans SQL et MongoDB, synchronisées en temps réel.

```
┌────────────────────────────────────────────────────────────────┐
│              Dual Database Architecture                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [Application]                                                 │
│       │                                                        │
│       ├──────────────┬─────────────────────────────┐           │
│       ↓              ↓                             ↓           │
│  ┌─────────┐   ┌──────────┐                ┌──────────┐        │
│  │ Writes  │   │  Reads   │                │Analytics │        │
│  │   ↓     │   │    ↓     │                │    ↓     │        │
│  │PostgreSQL   │ MongoDB  │                │ MongoDB  │        │
│  │(primary)│   │(replica) │                │(aggregate)        │
│  └────┬────┘   └────▲─────┘                └────▲─────┘        │
│       │             │                           │              │
│       └─────▶ [CDC: Debezium] ──────────────────┘              │
│                     │                                          │
│              [Kafka Topics]                                    │
│                     │                                          │
│              [Transformations]                                 │
│                                                                │
│  Sync modes:                                                   │
│  • Near real-time (< 1s lag)                                   │
│  • Bidirectional possible                                      │
│  • Schema transformation in-flight                             │
└────────────────────────────────────────────────────────────────┘
```

### Architecture 3 : CQRS (Command Query Responsibility Segregation)

**Principe** : Séparation write model (SQL) et read model (MongoDB).

```
┌────────────────────────────────────────────────────────────────┐
│                    CQRS Architecture                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [Commands]                         [Queries]                  │
│      ↓                                  ↑                      │
│  ┌─────────────┐                   ┌────────────┐              │
│  │  Write Side │                   │ Read Side  │              │
│  │             │                   │            │              │
│  │ PostgreSQL  │                   │  MongoDB   │              │
│  │ • Normalized│                   │• Denormalized             │
│  │ • ACID      │                   │• Optimized │              │
│  │ • Source of │                   │  for reads │              │
│  │   Truth     │                   │• Projections              │
│  └──────┬──────┘                   └─────▲──────┘              │
│         │                                │                     │
│         │                                │                     │
│         └───▶ [Event Bus: Kafka] ────────┘                     │
│                      │                                         │
│              [Event Handlers]                                  │
│              • Update read models                              │
│              • Trigger side effects                            │
│                                                                │
│  Benefits:                                                     │
│  • Scalabilité reads indépendante                              │
│  • Optimisation spécifique read/write                          │
│  • Eventual consistency acceptable                             │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Synchronisation Bidirectionnelle

### Stratégie 1 : CDC (Change Data Capture)

**PostgreSQL → MongoDB avec Debezium**

**docker-compose.yml**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: production
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mongodb:
    image: mongo:7.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  debezium:
    image: debezium/connect:2.5
    depends_on:
      - kafka
      - postgres
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-group
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses

volumes:
  postgres_data:
  mongo_data:
```

**Configuration Debezium Connector**

```json
{
  "name": "postgres-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "plugin.name": "pgoutput",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "postgres",
    "database.dbname": "production",
    "database.server.name": "postgres-server",
    "table.include.list": "public.orders,public.customers,public.products",
    "publication.autocreate.mode": "filtered",
    "topic.prefix": "postgres",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```

**Consumer Kafka → MongoDB (Python)**

```python
# kafka_mongodb_sync.py
from kafka import KafkaConsumer
from pymongo import MongoClient, UpdateOne
import json
import logging
from datetime import datetime

class KafkaMongoDBSync:
    def __init__(self, kafka_brokers, mongo_uri):
        self.consumer = KafkaConsumer(
            'postgres.public.orders',
            'postgres.public.customers',
            'postgres.public.products',
            bootstrap_servers=kafka_brokers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=True,
            group_id='mongodb-sync-group'
        )

        self.mongo_client = MongoClient(mongo_uri)
        self.db = self.mongo_client['production']

        self.logger = logging.getLogger(__name__)

        # Mapping PostgreSQL tables → MongoDB collections
        self.collection_mapping = {
            'postgres.public.orders': 'orders',
            'postgres.public.customers': 'customers',
            'postgres.public.products': 'products'
        }

        # Transformation rules
        self.transformations = {
            'orders': self.transform_order,
            'customers': self.transform_customer,
            'products': self.transform_product
        }

    def start(self):
        """Start consuming and syncing"""
        self.logger.info("Starting Kafka → MongoDB sync")

        for message in self.consumer:
            try:
                self.process_message(message)
            except Exception as e:
                self.logger.error(f"Error processing message: {e}")

    def process_message(self, message):
        """Process single Kafka message"""
        topic = message.topic
        value = message.value

        if not value:
            self.logger.warning(f"Empty message on topic {topic}")
            return

        # Get collection name
        collection_name = self.collection_mapping.get(topic)
        if not collection_name:
            self.logger.warning(f"No mapping for topic {topic}")
            return

        collection = self.db[collection_name]

        # Extract operation
        operation = value.get('__op')  # c=create, u=update, d=delete

        if operation == 'd':
            # Delete
            doc_id = value.get('before', {}).get('id')
            if doc_id:
                collection.delete_one({'_id': doc_id})
                self.logger.info(f"Deleted {collection_name} id={doc_id}")

        else:
            # Insert or Update
            doc = value.get('after', {})

            if not doc:
                return

            # Transform document
            transformation_func = self.transformations.get(collection_name)
            if transformation_func:
                doc = transformation_func(doc)

            # Add sync metadata
            doc['_synced_at'] = datetime.utcnow()
            doc['_source'] = 'postgresql'

            # Upsert
            doc_id = doc.get('id')
            if doc_id:
                doc['_id'] = doc_id
                collection.replace_one(
                    {'_id': doc_id},
                    doc,
                    upsert=True
                )
                self.logger.info(f"Upserted {collection_name} id={doc_id}")

    def transform_order(self, doc):
        """Transform order document from SQL to MongoDB format"""
        # Rename fields
        if 'customer_id' in doc:
            doc['customer_id'] = str(doc['customer_id'])

        # Parse JSON fields (if stored as JSON in PostgreSQL)
        if 'items' in doc and isinstance(doc['items'], str):
            doc['items'] = json.loads(doc['items'])

        # Convert timestamps
        for field in ['created_at', 'updated_at', 'order_date']:
            if field in doc and isinstance(doc[field], str):
                doc[field] = datetime.fromisoformat(doc[field].replace('Z', '+00:00'))

        return doc

    def transform_customer(self, doc):
        """Transform customer document"""
        # Nested structure
        if 'address_line1' in doc:
            doc['address'] = {
                'line1': doc.pop('address_line1', None),
                'line2': doc.pop('address_line2', None),
                'city': doc.pop('city', None),
                'state': doc.pop('state', None),
                'zip': doc.pop('zip_code', None),
                'country': doc.pop('country', None)
            }

        return doc

    def transform_product(self, doc):
        """Transform product document"""
        # Parse attributes JSON
        if 'attributes' in doc and isinstance(doc['attributes'], str):
            doc['attributes'] = json.loads(doc['attributes'])

        return doc

# Usage
if __name__ == '__main__':
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )

    sync = KafkaMongoDBSync(
        kafka_brokers=['localhost:9092'],
        mongo_uri='mongodb://admin:password@localhost:27017'
    )

    sync.start()
```

### Stratégie 2 : Application-Level Dual Writes

**Pattern: Write-Through à deux bases**

```python
# dual_write_repository.py
from pymongo import MongoClient
import psycopg2
from psycopg2.extras import RealDictCursor
import json
from datetime import datetime
from typing import Dict, Any
import logging

class DualWriteRepository:
    """Repository pattern avec dual writes SQL + MongoDB"""

    def __init__(self, postgres_conn_str, mongo_uri):
        # PostgreSQL connection
        self.pg_conn = psycopg2.connect(postgres_conn_str)
        self.pg_conn.autocommit = False

        # MongoDB connection
        self.mongo_client = MongoClient(mongo_uri)
        self.mongo_db = self.mongo_client['production']

        self.logger = logging.getLogger(__name__)

    def create_order(self, order_data: Dict[str, Any]) -> str:
        """
        Create order in both PostgreSQL and MongoDB

        Strategy: PostgreSQL is primary, MongoDB is secondary
        If MongoDB fails, log error but don't rollback PostgreSQL
        """
        order_id = order_data.get('order_id')

        try:
            # Step 1: Write to PostgreSQL (ACID transaction)
            with self.pg_conn.cursor() as cursor:
                cursor.execute("""
                    INSERT INTO orders (
                        order_id, customer_id, order_date,
                        status, total, items, created_at
                    ) VALUES (
                        %(order_id)s, %(customer_id)s, %(order_date)s,
                        %(status)s, %(total)s, %(items)s, %(created_at)s
                    )
                """, {
                    'order_id': order_id,
                    'customer_id': order_data['customer_id'],
                    'order_date': order_data['order_date'],
                    'status': order_data['status'],
                    'total': order_data['total'],
                    'items': json.dumps(order_data['items']),
                    'created_at': datetime.utcnow()
                })

            self.pg_conn.commit()
            self.logger.info(f"Order {order_id} created in PostgreSQL")

            # Step 2: Write to MongoDB (best effort)
            try:
                mongo_doc = self.transform_for_mongodb(order_data)
                mongo_doc['_id'] = order_id
                mongo_doc['_source'] = 'application'
                mongo_doc['_synced_at'] = datetime.utcnow()

                self.mongo_db.orders.insert_one(mongo_doc)
                self.logger.info(f"Order {order_id} created in MongoDB")

            except Exception as e:
                # MongoDB write failed - log but don't fail request
                self.logger.error(f"MongoDB write failed for order {order_id}: {e}")

                # Queue for retry (e.g., publish to dead letter queue)
                self.queue_for_retry('create_order', order_data)

            return order_id

        except Exception as e:
            # PostgreSQL write failed - rollback and fail request
            self.pg_conn.rollback()
            self.logger.error(f"PostgreSQL write failed for order {order_id}: {e}")
            raise

    def update_order_status(self, order_id: str, new_status: str):
        """Update order status in both databases"""

        try:
            # PostgreSQL update
            with self.pg_conn.cursor() as cursor:
                cursor.execute("""
                    UPDATE orders
                    SET status = %s, updated_at = %s
                    WHERE order_id = %s
                """, (new_status, datetime.utcnow(), order_id))

            self.pg_conn.commit()

            # MongoDB update
            try:
                self.mongo_db.orders.update_one(
                    {'_id': order_id},
                    {
                        '$set': {
                            'status': new_status,
                            'updated_at': datetime.utcnow(),
                            '_synced_at': datetime.utcnow()
                        }
                    }
                )
            except Exception as e:
                self.logger.error(f"MongoDB update failed: {e}")
                self.queue_for_retry('update_order_status', {
                    'order_id': order_id,
                    'status': new_status
                })

        except Exception as e:
            self.pg_conn.rollback()
            raise

    def get_order(self, order_id: str, prefer_mongodb: bool = True) -> Dict:
        """
        Get order, preferring MongoDB for performance
        Fallback to PostgreSQL if MongoDB fails
        """

        if prefer_mongodb:
            try:
                doc = self.mongo_db.orders.find_one({'_id': order_id})
                if doc:
                    self.logger.debug(f"Order {order_id} retrieved from MongoDB")
                    return doc
            except Exception as e:
                self.logger.warning(f"MongoDB read failed, falling back to PostgreSQL: {e}")

        # Fallback to PostgreSQL
        with self.pg_conn.cursor(cursor_factory=RealDictCursor) as cursor:
            cursor.execute("""
                SELECT * FROM orders WHERE order_id = %s
            """, (order_id,))

            row = cursor.fetchone()
            if row:
                self.logger.debug(f"Order {order_id} retrieved from PostgreSQL")
                return dict(row)

        return None

    def transform_for_mongodb(self, sql_data: Dict) -> Dict:
        """Transform SQL flat structure to MongoDB nested document"""

        mongo_doc = {
            'order_id': sql_data['order_id'],
            'customer_id': sql_data['customer_id'],
            'order_date': sql_data['order_date'],
            'status': sql_data['status'],
            'total': sql_data['total'],
            'items': sql_data['items'],  # Already list of dicts
            'timestamps': {
                'created_at': sql_data.get('created_at', datetime.utcnow()),
                'updated_at': sql_data.get('updated_at')
            }
        }

        return mongo_doc

    def queue_for_retry(self, operation: str, data: Dict):
        """Queue failed MongoDB writes for retry"""
        # Implementation: write to retry queue (e.g., Redis, Kafka DLQ)
        pass

    def reconcile_differences(self):
        """
        Periodic reconciliation job to fix divergences
        between PostgreSQL and MongoDB
        """

        with self.pg_conn.cursor(cursor_factory=RealDictCursor) as cursor:
            # Get all order IDs from PostgreSQL
            cursor.execute("SELECT order_id FROM orders")
            pg_order_ids = {row['order_id'] for row in cursor.fetchall()}

        # Get all order IDs from MongoDB
        mongo_order_ids = {
            doc['_id']
            for doc in self.mongo_db.orders.find({}, {'_id': 1})
        }

        # Find differences
        missing_in_mongo = pg_order_ids - mongo_order_ids
        missing_in_pg = mongo_order_ids - pg_order_ids

        self.logger.info(f"Reconciliation: {len(missing_in_mongo)} missing in MongoDB, "
                        f"{len(missing_in_pg)} missing in PostgreSQL")

        # Sync missing documents to MongoDB
        for order_id in missing_in_mongo:
            order_data = self.get_order(order_id, prefer_mongodb=False)
            if order_data:
                mongo_doc = self.transform_for_mongodb(order_data)
                mongo_doc['_id'] = order_id
                mongo_doc['_reconciled'] = True
                self.mongo_db.orders.insert_one(mongo_doc)

# Usage
repo = DualWriteRepository(
    postgres_conn_str='postgresql://user:pass@localhost/production',
    mongo_uri='mongodb://admin:password@localhost:27017'
)

# Create order
order = {
    'order_id': 'ORD-123',
    'customer_id': 'CUST-456',
    'order_date': datetime.utcnow(),
    'status': 'pending',
    'total': 99.99,
    'items': [
        {'product_id': 'PROD-1', 'quantity': 2, 'price': 49.99}
    ]
}

repo.create_order(order)

# Get order (prefer MongoDB)
retrieved_order = repo.get_order('ORD-123')
```

---

## 🌉 API Gateway Pattern

### Architecture avec Kong + PostgreSQL + MongoDB

```
┌────────────────────────────────────────────────────────────────┐
│              API Gateway Architecture                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [Mobile/Web Clients]                                          │
│           ↓                                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                Kong API Gateway                          │  │
│  │  • Authentication (OAuth2, JWT)                          │  │
│  │  • Rate limiting                                         │  │
│  │  • Request routing                                       │  │
│  │  • Response caching                                      │  │
│  │  • Analytics                                             │  │
│  └────────────┬─────────────────────────────────────┬───────┘  │
│               │                                     │          │
│       ┌───────┴────────┐                   ┌────────┴──────┐   │
│       ↓                ↓                   ↓               ↓   │
│  ┌──────────┐   ┌────────────┐      ┌──────────┐  ┌──────────┐ │
│  │ Orders   │   │  Products  │      │  Users   │  │Analytics │ │
│  │ Service  │   │  Service   │      │ Service  │  │ Service  │ │
│  └────┬─────┘   └─────┬──────┘      └────┬─────┘  └────┬─────┘ │
│       │               │                  │             │       │
│       ↓               ↓                  ↓             ↓       │
│  ┌─────────┐    ┌──────────┐        ┌────────┐   ┌──────────┐  │
│  │PostgreSQL│   │ MongoDB  │        │ Postgres│  │ MongoDB  │  │
│  └─────────┘    └──────────┘        └────────┘   └──────────┘  │
│                                                                │
│  Routing Rules:                                                │
│  POST   /orders      → Orders Service (PostgreSQL)             │
│  GET    /products/*  → Products Service (MongoDB)              │
│  GET    /users/*     → Users Service (PostgreSQL)              │
│  GET    /analytics/* → Analytics Service (MongoDB)             │
└────────────────────────────────────────────────────────────────┘
```

**Configuration Kong (declarative)**

```yaml
# kong.yml
_format_version: "3.0"

services:
  - name: orders-service
    url: http://orders-service:8000
    routes:
      - name: orders-route
        paths:
          - /orders
        methods:
          - POST
          - GET
          - PUT
          - DELETE
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: jwt
      - name: cors

  - name: products-service
    url: http://products-service:8000
    routes:
      - name: products-route
        paths:
          - /products
        methods:
          - GET
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          minute: 1000
          policy: local
      - name: proxy-cache
        config:
          strategy: memory
          cache_ttl: 300
          content_type:
            - application/json
      - name: cors

  - name: users-service
    url: http://users-service:8000
    routes:
      - name: users-route
        paths:
          - /users
        methods:
          - GET
          - POST
          - PUT
    plugins:
      - name: jwt
      - name: acl
        config:
          allow:
            - admin
            - user
      - name: cors

  - name: analytics-service
    url: http://analytics-service:8000
    routes:
      - name: analytics-route
        paths:
          - /analytics
        methods:
          - GET
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 50
          policy: local
      - name: cors

plugins:
  - name: prometheus
  - name: request-transformer
    config:
      add:
        headers:
          - X-Gateway-Version:1.0
```

---

## 🔀 Patterns d'Intégration Avancés

### Pattern 1 : Saga Pattern (Distributed Transaction)

**Orchestration-based Saga avec MongoDB + PostgreSQL**

```python
# saga_orchestrator.py
from enum import Enum
from typing import Dict, List
from pymongo import MongoClient
import psycopg2
import requests
from datetime import datetime
import logging

class SagaStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    COMPENSATING = "compensating"
    COMPENSATED = "compensated"

class OrderSaga:
    """
    Saga orchestrator pour création commande

    Steps:
    1. Reserve inventory (MongoDB)
    2. Process payment (PostgreSQL)
    3. Create order (PostgreSQL)
    4. Update customer stats (MongoDB)

    Compensation (rollback):
    4c. Revert customer stats
    3c. Cancel order
    2c. Refund payment
    1c. Release inventory
    """

    def __init__(self, mongo_uri, postgres_conn_str):
        self.mongo_client = MongoClient(mongo_uri)
        self.mongo_db = self.mongo_client['production']

        self.pg_conn = psycopg2.connect(postgres_conn_str)

        self.logger = logging.getLogger(__name__)

    def execute_saga(self, order_data: Dict) -> Dict:
        """Execute saga with automatic compensation on failure"""

        saga_id = f"saga_{datetime.utcnow().timestamp()}"

        # Initialize saga state
        saga_state = {
            '_id': saga_id,
            'status': SagaStatus.PENDING.value,
            'order_data': order_data,
            'completed_steps': [],
            'current_step': None,
            'created_at': datetime.utcnow(),
            'updated_at': datetime.utcnow()
        }

        self.mongo_db.sagas.insert_one(saga_state)

        try:
            # Step 1: Reserve inventory
            self.update_saga_status(saga_id, SagaStatus.IN_PROGRESS, 'reserve_inventory')
            inventory_result = self.reserve_inventory(order_data['items'])
            self.add_completed_step(saga_id, 'reserve_inventory', inventory_result)

            # Step 2: Process payment
            self.update_saga_status(saga_id, SagaStatus.IN_PROGRESS, 'process_payment')
            payment_result = self.process_payment(
                order_data['customer_id'],
                order_data['total']
            )
            self.add_completed_step(saga_id, 'process_payment', payment_result)

            # Step 3: Create order
            self.update_saga_status(saga_id, SagaStatus.IN_PROGRESS, 'create_order')
            order_result = self.create_order(order_data, payment_result['transaction_id'])
            self.add_completed_step(saga_id, 'create_order', order_result)

            # Step 4: Update customer stats
            self.update_saga_status(saga_id, SagaStatus.IN_PROGRESS, 'update_customer_stats')
            stats_result = self.update_customer_stats(
                order_data['customer_id'],
                order_data['total']
            )
            self.add_completed_step(saga_id, 'update_customer_stats', stats_result)

            # Success
            self.update_saga_status(saga_id, SagaStatus.COMPLETED)

            return {
                'saga_id': saga_id,
                'status': 'success',
                'order_id': order_result['order_id']
            }

        except Exception as e:
            self.logger.error(f"Saga {saga_id} failed: {e}")

            # Trigger compensation
            self.update_saga_status(saga_id, SagaStatus.COMPENSATING)
            self.compensate(saga_id)

            return {
                'saga_id': saga_id,
                'status': 'failed',
                'error': str(e)
            }

    def reserve_inventory(self, items: List[Dict]) -> Dict:
        """Reserve inventory in MongoDB"""

        reserved_items = []

        for item in items:
            result = self.mongo_db.inventory.update_one(
                {
                    'product_id': item['product_id'],
                    'available_quantity': {'$gte': item['quantity']}
                },
                {
                    '$inc': {
                        'available_quantity': -item['quantity'],
                        'reserved_quantity': item['quantity']
                    }
                }
            )

            if result.modified_count == 0:
                raise Exception(f"Insufficient inventory for product {item['product_id']}")

            reserved_items.append(item)

        return {'reserved_items': reserved_items}

    def process_payment(self, customer_id: str, amount: float) -> Dict:
        """Process payment in PostgreSQL"""

        with self.pg_conn.cursor() as cursor:
            # Check customer balance (simplified)
            cursor.execute("""
                SELECT balance FROM customers WHERE customer_id = %s
            """, (customer_id,))

            row = cursor.fetchone()
            if not row or row[0] < amount:
                raise Exception(f"Insufficient balance for customer {customer_id}")

            # Deduct amount
            cursor.execute("""
                UPDATE customers
                SET balance = balance - %s
                WHERE customer_id = %s
            """, (amount, customer_id))

            # Create transaction record
            cursor.execute("""
                INSERT INTO transactions (customer_id, amount, type, created_at)
                VALUES (%s, %s, 'payment', %s)
                RETURNING transaction_id
            """, (customer_id, amount, datetime.utcnow()))

            transaction_id = cursor.fetchone()[0]

        self.pg_conn.commit()

        return {'transaction_id': transaction_id, 'amount': amount}

    def create_order(self, order_data: Dict, transaction_id: str) -> Dict:
        """Create order in PostgreSQL"""

        with self.pg_conn.cursor() as cursor:
            cursor.execute("""
                INSERT INTO orders (
                    customer_id, transaction_id, status,
                    total, items, created_at
                )
                VALUES (%s, %s, %s, %s, %s, %s)
                RETURNING order_id
            """, (
                order_data['customer_id'],
                transaction_id,
                'pending',
                order_data['total'],
                psycopg2.extras.Json(order_data['items']),
                datetime.utcnow()
            ))

            order_id = cursor.fetchone()[0]

        self.pg_conn.commit()

        return {'order_id': order_id}

    def update_customer_stats(self, customer_id: str, amount: float) -> Dict:
        """Update customer statistics in MongoDB"""

        result = self.mongo_db.customer_stats.update_one(
            {'customer_id': customer_id},
            {
                '$inc': {
                    'total_orders': 1,
                    'total_spent': amount
                },
                '$set': {
                    'last_order_date': datetime.utcnow()
                }
            },
            upsert=True
        )

        return {'updated': True}

    def compensate(self, saga_id: str):
        """Execute compensation (rollback) for failed saga"""

        saga = self.mongo_db.sagas.find_one({'_id': saga_id})
        completed_steps = saga.get('completed_steps', [])

        # Reverse order compensation
        for step in reversed(completed_steps):
            step_name = step['step']
            step_data = step['data']

            try:
                if step_name == 'reserve_inventory':
                    self.compensate_inventory(step_data)

                elif step_name == 'process_payment':
                    self.compensate_payment(step_data)

                elif step_name == 'create_order':
                    self.compensate_order(step_data)

                elif step_name == 'update_customer_stats':
                    self.compensate_customer_stats(saga['order_data'], step_data)

                self.logger.info(f"Compensated step {step_name}")

            except Exception as e:
                self.logger.error(f"Compensation failed for {step_name}: {e}")

        self.update_saga_status(saga_id, SagaStatus.COMPENSATED)

    def compensate_inventory(self, step_data: Dict):
        """Rollback inventory reservation"""
        for item in step_data['reserved_items']:
            self.mongo_db.inventory.update_one(
                {'product_id': item['product_id']},
                {
                    '$inc': {
                        'available_quantity': item['quantity'],
                        'reserved_quantity': -item['quantity']
                    }
                }
            )

    def compensate_payment(self, step_data: Dict):
        """Refund payment"""
        # Implementation omitted for brevity
        pass

    def compensate_order(self, step_data: Dict):
        """Cancel order"""
        # Implementation omitted for brevity
        pass

    def compensate_customer_stats(self, order_data: Dict, step_data: Dict):
        """Revert customer statistics"""
        self.mongo_db.customer_stats.update_one(
            {'customer_id': order_data['customer_id']},
            {
                '$inc': {
                    'total_orders': -1,
                    'total_spent': -order_data['total']
                }
            }
        )

    def update_saga_status(self, saga_id: str, status: SagaStatus, current_step: str = None):
        """Update saga status"""
        update = {
            '$set': {
                'status': status.value,
                'updated_at': datetime.utcnow()
            }
        }

        if current_step:
            update['$set']['current_step'] = current_step

        self.mongo_db.sagas.update_one({'_id': saga_id}, update)

    def add_completed_step(self, saga_id: str, step_name: str, step_data: Dict):
        """Record completed step"""
        self.mongo_db.sagas.update_one(
            {'_id': saga_id},
            {
                '$push': {
                    'completed_steps': {
                        'step': step_name,
                        'data': step_data,
                        'completed_at': datetime.utcnow()
                    }
                }
            }
        )

# Usage
saga = OrderSaga(
    mongo_uri='mongodb://localhost:27017',
    postgres_conn_str='postgresql://user:pass@localhost/production'
)

order_data = {
    'customer_id': 'CUST-123',
    'items': [
        {'product_id': 'PROD-1', 'quantity': 2, 'price': 29.99}
    ],
    'total': 59.98
}

result = saga.execute_saga(order_data)
print(result)
```

### Pattern 2 : Outbox Pattern

**Reliable event publishing sans distributed transactions**

```python
# outbox_pattern.py
from pymongo import MongoClient
import psycopg2
from datetime import datetime
import json
import time
from kafka import KafkaProducer

class OutboxPublisher:
    """
    Outbox pattern implementation

    Ensures reliable event publishing from PostgreSQL → Kafka
    without distributed transactions
    """

    def __init__(self, postgres_conn_str, kafka_brokers):
        self.pg_conn = psycopg2.connect(postgres_conn_str)
        self.producer = KafkaProducer(
            bootstrap_servers=kafka_brokers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    def create_order_with_outbox(self, order_data):
        """
        Create order and write event to outbox table
        in same transaction
        """

        with self.pg_conn.cursor() as cursor:
            # Business operation: Create order
            cursor.execute("""
                INSERT INTO orders (customer_id, total, items, status, created_at)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING order_id
            """, (
                order_data['customer_id'],
                order_data['total'],
                psycopg2.extras.Json(order_data['items']),
                'pending',
                datetime.utcnow()
            ))

            order_id = cursor.fetchone()[0]

            # Write event to outbox (same transaction)
            event = {
                'event_type': 'OrderCreated',
                'aggregate_id': order_id,
                'aggregate_type': 'Order',
                'payload': {
                    'order_id': order_id,
                    'customer_id': order_data['customer_id'],
                    'total': order_data['total'],
                    'items': order_data['items']
                }
            }

            cursor.execute("""
                INSERT INTO outbox (
                    event_type, aggregate_id, aggregate_type,
                    payload, created_at, published
                )
                VALUES (%s, %s, %s, %s, %s, %s)
            """, (
                event['event_type'],
                event['aggregate_id'],
                event['aggregate_type'],
                psycopg2.extras.Json(event['payload']),
                datetime.utcnow(),
                False
            ))

        # Commit both operations atomically
        self.pg_conn.commit()

        return order_id

    def publish_outbox_events(self):
        """
        Separate process: Read unpublished events and publish to Kafka
        """

        while True:
            try:
                # Fetch unpublished events
                with self.pg_conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cursor:
                    cursor.execute("""
                        SELECT * FROM outbox
                        WHERE published = FALSE
                        ORDER BY created_at
                        LIMIT 100
                        FOR UPDATE SKIP LOCKED
                    """)

                    events = cursor.fetchall()

                # Publish to Kafka
                for event in events:
                    topic = f"{event['aggregate_type']}.{event['event_type']}"

                    self.producer.send(
                        topic,
                        key=event['aggregate_id'].encode('utf-8'),
                        value=event['payload']
                    )

                    # Mark as published
                    with self.pg_conn.cursor() as cursor:
                        cursor.execute("""
                            UPDATE outbox
                            SET published = TRUE, published_at = %s
                            WHERE id = %s
                        """, (datetime.utcnow(), event['id']))

                    self.pg_conn.commit()

                self.producer.flush()

                if events:
                    print(f"Published {len(events)} events")

                time.sleep(1)  # Poll interval

            except Exception as e:
                print(f"Error publishing events: {e}")
                time.sleep(5)
```

---

## 📊 Scénario Réel : FinTech Platform (36 mois)

**Contexte**
- Plateforme paiements B2B
- Transactions financières : PostgreSQL (ACID critique)
- Analytics et audit : MongoDB (volume + requêtes complexes)
- 100M transactions/mois
- Conformité stricte (PCI-DSS, SOC2)

### Architecture Hybride

```
┌─────────────────────────────────────────────────────────────────┐
│           FinTech Hybrid Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Payment API]                                                  │
│       │                                                         │
│       ↓                                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           Transaction Service                            │   │
│  │  • ACID transactions                                     │   │
│  │  • PostgreSQL (primary)                                  │   │
│  │  • MongoDB (analytics replica)                           │   │
│  └────────────┬────────────────────────────────────┬────────┘   │
│               │                                    │            │
│               ↓                                    ↓            │
│  ┌──────────────────┐                    ┌──────────────────┐   │
│  │   PostgreSQL     │                    │     MongoDB      │   │
│  │                  │                    │                  │   │
│  │ • transactions   │──CDC (Debezium)──▶ │ • transactions   │   │
│  │ • accounts       │                    │   (denormalized) │   │
│  │ • ledger_entries │                    │ • audit_logs     │   │
│  │                  │                    │ • analytics_agg  │   │
│  │ Retention: 7y    │                    │ Retention: ∞     │   │
│  └──────────────────┘                    └──────────────────┘   │
│         │                                         │             │
│         │                                         ↓             │
│         │                                 [BI Tools]            │
│         │                                 [Compliance Reports]  │
│         │                                                       │
│         ↓                                                       │
│  [Archive (S3)]                                                 │
│  • Cold storage (>7y)                                           │
│  • Parquet format                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Décisions Architecture

**PostgreSQL pour :**
- ✅ Transactions financières (ACID absolu)
- ✅ Account balances (strong consistency)
- ✅ Ledger entries (double-entry bookkeeping)
- ✅ Compliance audit trail (immutabilité)

**MongoDB pour :**
- ✅ Real-time analytics (dashboards)
- ✅ Complex aggregations (reporting)
- ✅ Audit logs enrichis (full documents)
- ✅ Historical data (unlimited retention)
- ✅ Flexible schema (evolving metadata)

### Pipeline CDC Production

**Configuration Debezium**
```json
{
  "name": "fintech-postgres-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-primary.internal",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "${file:/secrets/db-password}",
    "database.dbname": "fintech_production",
    "database.server.name": "fintech",
    "table.include.list": "public.transactions,public.accounts,public.ledger_entries",
    "plugin.name": "pgoutput",
    "publication.autocreate.mode": "filtered",
    "slot.name": "debezium_slot",
    "heartbeat.interval.ms": "10000",
    "snapshot.mode": "initial",
    "snapshot.locking.mode": "none",
    "decimal.handling.mode": "precise",
    "time.precision.mode": "connect",
    "include.schema.changes": "false",
    "transforms": "unwrap,addFields",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.addFields.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addFields.static.field": "cdc_source",
    "transforms.addFields.static.value": "postgresql",
    "transforms.addFields.timestamp.field": "cdc_timestamp"
  }
}
```

### Métriques Production

**Volume**
- Transactions : 100M/mois (3.3M/jour)
- CDC latency : < 500ms (P95)
- MongoDB sync : 99.99% consistency
- PostgreSQL → MongoDB lag : < 1 second

**Performance**
- PostgreSQL writes : 5K TPS
- MongoDB aggregations : < 100ms (P95)
- Dashboard queries : < 200ms (P95)

**Reliability**
- Uptime : 99.99% (4.38 min downtime/mois)
- Data loss : 0 (zero tolerance)
- CDC failures : Auto-recovery < 30s

### Challenges

**Challenge 1 : CDC lag spikes**
- **Symptôme** : Lag monte à 30 secondes lors bulk operations
- **Cause** : Nightly reconciliation jobs (millions updates)
- **Solution** : Rate limiting + separate CDC slot

**Challenge 2 : Schema evolution**
- **Symptôme** : Breaking changes cassent consommateurs MongoDB
- **Cause** : Pas de versioning
- **Solution** : Schema registry + backward compatibility tests

**Challenge 3 : Data divergence**
- **Symptôme** : 0.01% divergence détectée en audit
- **Cause** : Race conditions en CDC
- **Solution** : Reconciliation job quotidien + alerting

---

## 🎯 Bonnes Pratiques

### 1. Choix de la Base par Use Case

```python
# Decision matrix implementation
class DatabaseSelector:
    """Helper to select appropriate database"""

    @staticmethod
    def select_database(use_case_params):
        """
        Select database based on use case requirements

        Params:
        - acid_required: bool
        - schema_flexible: bool
        - read_heavy: bool
        - write_heavy: bool
        - analytics: bool
        - volume_tb: float
        """

        score_postgres = 0
        score_mongodb = 0

        if use_case_params.get('acid_required'):
            score_postgres += 10

        if use_case_params.get('schema_flexible'):
            score_mongodb += 8

        if use_case_params.get('analytics'):
            score_mongodb += 7

        if use_case_params.get('read_heavy'):
            score_mongodb += 5

        if use_case_params.get('volume_tb', 0) > 5:
            score_mongodb += 6

        if score_postgres > score_mongodb:
            return 'PostgreSQL'
        elif score_mongodb > score_postgres:
            return 'MongoDB'
        else:
            return 'Hybrid (both)'

# Examples
print(DatabaseSelector.select_database({
    'acid_required': True,
    'schema_flexible': False,
    'analytics': False
}))
# Output: PostgreSQL

print(DatabaseSelector.select_database({
    'acid_required': False,
    'schema_flexible': True,
    'analytics': True,
    'volume_tb': 10
}))
# Output: MongoDB
```

### 2. Monitoring Hybride

```python
# hybrid_monitoring.py
from prometheus_client import Gauge, Counter, Histogram
import time

# Metrics
sync_lag_gauge = Gauge('cdc_sync_lag_seconds', 'CDC sync lag', ['source', 'target'])
sync_errors_counter = Counter('cdc_sync_errors_total', 'CDC sync errors')
query_latency_histogram = Histogram('query_latency_seconds', 'Query latency', ['database'])

class HybridMonitoring:
    def __init__(self, pg_conn, mongo_client):
        self.pg_conn = pg_conn
        self.mongo_client = mongo_client

    def measure_sync_lag(self):
        """Measure lag between PostgreSQL and MongoDB"""

        # Get latest timestamp from PostgreSQL
        with self.pg_conn.cursor() as cursor:
            cursor.execute("SELECT MAX(updated_at) FROM transactions")
            pg_latest = cursor.fetchone()[0]

        # Get latest synced timestamp from MongoDB
        mongo_latest = self.mongo_client['production'].transactions.find_one(
            sort=[('updated_at', -1)]
        )['updated_at']

        # Calculate lag
        lag_seconds = (pg_latest - mongo_latest).total_seconds()

        sync_lag_gauge.labels(source='postgresql', target='mongodb').set(lag_seconds)

        return lag_seconds

    def monitor_query_performance(self):
        """Monitor query performance on both databases"""

        # PostgreSQL query
        start = time.time()
        with self.pg_conn.cursor() as cursor:
            cursor.execute("SELECT COUNT(*) FROM transactions WHERE status = 'completed'")
            cursor.fetchone()
        pg_duration = time.time() - start
        query_latency_histogram.labels(database='postgresql').observe(pg_duration)

        # MongoDB query
        start = time.time()
        count = self.mongo_client['production'].transactions.count_documents({
            'status': 'completed'
        })
        mongo_duration = time.time() - start
        query_latency_histogram.labels(database='mongodb').observe(mongo_duration)
```

### 3. Testing Hybride

```python
# hybrid_integration_test.py
import pytest
from pymongo import MongoClient
import psycopg2
import time

class TestHybridSystem:

    @pytest.fixture
    def setup_databases(self):
        # Setup test databases
        pg_conn = psycopg2.connect("postgresql://localhost/test_db")
        mongo_client = MongoClient("mongodb://localhost:27017/test_db")

        yield pg_conn, mongo_client

        # Cleanup
        pg_conn.close()
        mongo_client.close()

    def test_dual_write_consistency(self, setup_databases):
        """Test that dual writes maintain consistency"""
        pg_conn, mongo_client = setup_databases

        # Write to PostgreSQL
        with pg_conn.cursor() as cursor:
            cursor.execute("""
                INSERT INTO orders (order_id, customer_id, total)
                VALUES ('TEST-123', 'CUST-456', 99.99)
            """)
        pg_conn.commit()

        # Write to MongoDB
        mongo_client['test_db'].orders.insert_one({
            '_id': 'TEST-123',
            'customer_id': 'CUST-456',
            'total': 99.99
        })

        # Verify consistency
        with pg_conn.cursor() as cursor:
            cursor.execute("SELECT * FROM orders WHERE order_id = 'TEST-123'")
            pg_order = cursor.fetchone()

        mongo_order = mongo_client['test_db'].orders.find_one({'_id': 'TEST-123'})

        assert pg_order[1] == mongo_order['customer_id']
        assert pg_order[2] == mongo_order['total']

    def test_cdc_propagation(self, setup_databases):
        """Test that CDC propagates changes correctly"""
        pg_conn, mongo_client = setup_databases

        # Write to PostgreSQL
        with pg_conn.cursor() as cursor:
            cursor.execute("""
                INSERT INTO orders (order_id, status)
                VALUES ('TEST-456', 'pending')
            """)
        pg_conn.commit()

        # Wait for CDC propagation
        time.sleep(2)

        # Verify MongoDB has the change
        mongo_order = mongo_client['test_db'].orders.find_one({'_id': 'TEST-456'})
        assert mongo_order is not None
        assert mongo_order['status'] == 'pending'
```

---

## 📚 Checklist Coexistence SQL + MongoDB

**Architecture**
- [ ] Use cases documentés (SQL vs MongoDB vs Hybrid)
- [ ] Bounded contexts définis
- [ ] Data flow diagrams créés
- [ ] Migration strategy (phased)
- [ ] Rollback plan

**Synchronisation**
- [ ] CDC configuré (Debezium ou custom)
- [ ] Transformation pipeline testé
- [ ] Lag monitoring actif
- [ ] Reconciliation jobs schedulés
- [ ] Dead letter queue configurée

**Application**
- [ ] Repository pattern implémenté
- [ ] Dual writes (si applicable)
- [ ] Fallback logic (si MongoDB down)
- [ ] Error handling robuste
- [ ] Retry logic

**Data Consistency**
- [ ] Consistency model défini (strong vs eventual)
- [ ] Validation tests automatisés
- [ ] Reconciliation quotidienne
- [ ] Divergence alerting
- [ ] Manual reconciliation process

**Performance**
- [ ] Indexes optimisés (both databases)
- [ ] Query patterns analysés
- [ ] Caching strategy (Redis)
- [ ] Load testing effectué
- [ ] Scaling strategy

**Monitoring**
- [ ] Sync lag metrics
- [ ] Query performance (both DBs)
- [ ] Error rates tracking
- [ ] Dashboards créés
- [ ] Alerting configuré

**Operations**
- [ ] Runbooks incidents
- [ ] On-call training
- [ ] Backup strategy (both DBs)
- [ ] Disaster recovery tested
- [ ] Documentation complète

---

## 🎓 Conclusion

La coexistence SQL + MongoDB est une réalité dans la plupart des organisations modernes. Points clés :

**Quand choisir Polyglot Persistence ?**
- ✅ Use cases diversifiés (ACID + Analytics)
- ✅ Migration progressive nécessaire
- ✅ Équipes avec expertises différentes
- ✅ Contraintes réglementaires mixtes

**Patterns recommandés**
- Bounded Context Segregation (microservices)
- CQRS (write SQL, read MongoDB)
- CDC Pipeline (synchronisation temps réel)
- API Gateway (abstraction)

**Challenges principaux**
- Data consistency (eventual vs strong)
- Operational complexity (2 databases)
- Team expertise (formation)
- Monitoring & observability

**ROI**
- Optimal database per use case
- Performance maximale
- Flexibility architecturale
- Migration risk minimisé

La coexistence SQL + MongoDB n'est pas un anti-pattern mais une stratégie pragmatique qui permet d'exploiter le meilleur des deux mondes tout en gérant la complexité de manière contrôlée.

---

**Fin du Chapitre 19 : Migration et Intégration**

Ce chapitre a couvert l'ensemble du spectre de migration et intégration MongoDB :
- 19.1 Vue d'ensemble et stratégies
- 19.2 Outils de migration
- 19.3 MongoDB Relational Migrator
- 19.4 Stratégies de migration incrémentale
- 19.5 Synchronisation bidirectionnelle
- 19.6 MongoDB Connector for BI
- 19.7 Intégration Apache Kafka
- 19.8 Intégration Apache Spark
- 19.9 ETL et Data Pipelines
- 19.10 Coexistence avec bases relationnelles


⏭️ [Cas d'Usage et Architectures](/20-cas-usage-architectures/README.md)
