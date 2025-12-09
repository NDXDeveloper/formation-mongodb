🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.9 ETL et Data Pipelines

## Introduction

Les pipelines ETL (Extract, Transform, Load) et ELT (Extract, Load, Transform) modernes sont l'épine dorsale des plateformes data. MongoDB s'intègre naturellement dans ces architectures grâce à sa flexibilité schéma et ses capacités d'agrégation puissantes. Cette section explore les patterns, outils et architectures pour construire des data pipelines robustes, scalables et maintenables en production.

---

## 🎯 Architectures ETL Modernes

### ETL vs ELT vs Reverse ETL

**ETL (Extract, Transform, Load)**
```
[Source DB] → [Transform (Spark/Python)] → [Target DB]
```
- Transformation avant chargement
- Idéal pour: Legacy systems, transformations complexes
- MongoDB: Souvent source ou cible

**ELT (Extract, Load, Transform)**
```
[Source DB] → [Data Lake/Warehouse] → [Transform (SQL/DBT)]
```
- Chargement brut puis transformation
- Idéal pour: Cloud data warehouses, données massives
- MongoDB: Excellente source via CDC

**Reverse ETL**
```
[Data Warehouse] → [Transform] → [Operational DB (MongoDB)]
```
- Data warehouse → applications opérationnelles
- Idéal pour: Enrichissement données, ML predictions
- MongoDB: Cible pour données enrichies

### Architecture Lambda

```
┌───────────────────────────────────────────────────────────────┐
│                  Lambda Architecture                          │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  [Data Sources]                                               │
│  • MongoDB (Change Streams)                                   │
│  • APIs                                                       │
│  • Files (S3)                                                 │
│       │                                                       │
│       ├──────────────────┬────────────────────────┐           │
│       ↓                  ↓                        ↓           │
│  ┌──────────┐    ┌──────────────┐         ┌─────────────┐     │
│  │  Batch   │    │   Streaming  │         │   Serving   │     │
│  │  Layer   │    │    Layer     │         │   Layer     │     │
│  │          │    │              │         │             │     │
│  │ (Spark)  │    │ (Kafka/Flink)│         │  (MongoDB)  │     │
│  │          │    │              │         │             │     │
│  │ Complete │    │  Real-time   │         │  Queries    │     │
│  │ history  │    │  incremental │         │             │     │
│  └────┬─────┘    └──────┬───────┘         └──────▲──────┘     │
│       │                 │                        │            │
│       │                 │                        │            │
│       └─────────────────┴────────────────────────┘            │
│                         │                                     │
│                  [Merge Results]                              │
│                         │                                     │
│                         ↓                                     │
│               [MongoDB Views/Aggregations]                    │
└───────────────────────────────────────────────────────────────┘
```

### Architecture Kappa (Streaming-First)

```
┌────────────────────────────────────────────────────────────────┐
│                  Kappa Architecture                            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [Data Sources] ──▶ [Kafka Topics] ──▶ [Stream Processing]     │
│                          │                      │              │
│                          │                      ↓              │
│                          │              [MongoDB Sink]         │
│                          │                      │              │
│                          │                      ↓              │
│                          └──▶ [Reprocess] ──▶ [Views]          │
│                                                                │
│  • Single pipeline pour batch et streaming                     │
│  • Reprocessing = replay events                                │
│  • MongoDB = Event Store + Materialized Views                  │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Apache Airflow pour Orchestration

### Architecture Airflow + MongoDB

```
┌───────────────────────────────────────────────────────────────┐
│           Airflow Orchestration Architecture                  │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              Airflow Components                          │ │
│  │                                                          │ │
│  │  • Scheduler (manages DAG execution)                     │ │
│  │  • Web UI (monitoring, logs)                             │ │
│  │  • Executor (Celery/Kubernetes)                          │ │
│  │  • Metadata DB (PostgreSQL)                              │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     │                                         │
│                     ↓                                         │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                   DAGs                                   │ │
│  │                                                          │ │
│  │  • extract_mongodb_dag (daily)                           │ │
│  │  • transform_spark_dag (daily)                           │ │
│  │  • load_mongodb_dag (daily)                              │ │
│  │  • data_quality_dag (hourly)                             │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     │                                         │
│                     ↓                                         │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              Execution                                   │ │
│  │                                                          │ │
│  │  Workers: 10x instances                                  │ │
│  │  • Python scripts                                        │ │
│  │  • Spark jobs                                            │ │
│  │  • MongoDB queries                                       │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### DAG MongoDB ETL Complet

**mongodb_etl_dag.py**

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.mongo.hooks.mongo import MongoHook
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.utils.dates import days_ago
from datetime import datetime, timedelta
import pandas as pd
import logging

# Configuration
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': days_ago(1),
    'email': ['data-alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=2)
}

dag = DAG(
    'mongodb_daily_etl',
    default_args=default_args,
    description='Daily ETL pipeline: MongoDB → Transform → MongoDB',
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False,
    max_active_runs=1,
    tags=['mongodb', 'etl', 'production']
)

# Task 1: Validate source data
def validate_source_data(**context):
    """Validate MongoDB source data quality"""
    hook = MongoHook(conn_id='mongodb_production')
    client = hook.get_conn()
    db = client['production']

    execution_date = context['execution_date']
    yesterday = execution_date - timedelta(days=1)

    # Check record count
    count = db.transactions.count_documents({
        'transaction_date': {
            '$gte': yesterday.replace(hour=0, minute=0, second=0),
            '$lt': execution_date.replace(hour=0, minute=0, second=0)
        }
    })

    logging.info(f"Source record count: {count}")

    if count == 0:
        raise ValueError("No records found for processing date")

    # Check data quality
    null_count = db.transactions.count_documents({
        'transaction_date': {'$gte': yesterday},
        '$or': [
            {'total': None},
            {'customer_id': None},
            {'status': None}
        ]
    })

    null_rate = null_count / count
    logging.info(f"Null rate: {null_rate:.2%}")

    if null_rate > 0.05:  # 5% threshold
        raise ValueError(f"Null rate too high: {null_rate:.2%}")

    # Push to XCom
    context['task_instance'].xcom_push(key='source_count', value=count)
    context['task_instance'].xcom_push(key='null_rate', value=null_rate)

    return count

validate_task = PythonOperator(
    task_id='validate_source_data',
    python_callable=validate_source_data,
    provide_context=True,
    dag=dag
)

# Task 2: Extract to staging
def extract_to_staging(**context):
    """Extract data from MongoDB to staging area"""
    hook = MongoHook(conn_id='mongodb_production')
    client = hook.get_conn()
    db = client['production']

    execution_date = context['execution_date']
    yesterday = execution_date - timedelta(days=1)

    # Extract with aggregation pipeline
    pipeline = [
        {
            '$match': {
                'transaction_date': {
                    '$gte': yesterday.replace(hour=0, minute=0, second=0),
                    '$lt': execution_date.replace(hour=0, minute=0, second=0)
                }
            }
        },
        {
            '$lookup': {
                'from': 'customers',
                'localField': 'customer_id',
                'foreignField': 'customer_id',
                'as': 'customer'
            }
        },
        {
            '$unwind': '$customer'
        },
        {
            '$project': {
                'transaction_id': 1,
                'transaction_date': 1,
                'customer_id': 1,
                'customer_name': '$customer.name',
                'customer_tier': '$customer.tier',
                'total': 1,
                'status': 1,
                'items': 1
            }
        },
        {
            '$out': 'staging_transactions'
        }
    ]

    result = list(db.transactions.aggregate(pipeline, allowDiskUse=True))

    # Verify staging
    staging_count = db.staging_transactions.count_documents({})
    logging.info(f"Staging record count: {staging_count}")

    context['task_instance'].xcom_push(key='staging_count', value=staging_count)

    return staging_count

extract_task = PythonOperator(
    task_id='extract_to_staging',
    python_callable=extract_to_staging,
    provide_context=True,
    dag=dag
)

# Task 3: Transform with Spark
transform_task = SparkSubmitOperator(
    task_id='transform_spark',
    application='/opt/airflow/spark/transform_transactions.py',
    conn_id='spark_default',
    conf={
        'spark.mongodb.read.connection.uri': 'mongodb://production/staging_transactions',
        'spark.mongodb.write.connection.uri': 'mongodb://production/transformed_transactions'
    },
    application_args=[
        '--execution-date', '{{ ds }}'
    ],
    executor_memory='16g',
    driver_memory='8g',
    num_executors=10,
    executor_cores=4,
    dag=dag
)

# Task 4: Load aggregated results
def load_aggregated_results(**context):
    """Load transformed data back to MongoDB"""
    hook = MongoHook(conn_id='mongodb_production')
    client = hook.get_conn()
    db = client['production']

    execution_date = context['execution_date']

    # Aggregate transformed data
    pipeline = [
        {
            '$match': {
                'processed_date': execution_date.date().isoformat()
            }
        },
        {
            '$group': {
                '_id': {
                    'date': '$transaction_date',
                    'customer_tier': '$customer_tier'
                },
                'transaction_count': {'$sum': 1},
                'total_revenue': {'$sum': '$total'},
                'avg_order_value': {'$avg': '$total'},
                'unique_customers': {'$addToSet': '$customer_id'}
            }
        },
        {
            '$project': {
                '_id': 0,
                'date': '$_id.date',
                'customer_tier': '$_id.customer_tier',
                'transaction_count': 1,
                'total_revenue': 1,
                'avg_order_value': 1,
                'unique_customers_count': {'$size': '$unique_customers'}
            }
        }
    ]

    aggregated = list(db.transformed_transactions.aggregate(pipeline))

    # Bulk write to analytics collection
    if aggregated:
        db.daily_analytics.insert_many(aggregated)
        logging.info(f"Loaded {len(aggregated)} aggregated records")

    context['task_instance'].xcom_push(key='aggregated_count', value=len(aggregated))

    return len(aggregated)

load_task = PythonOperator(
    task_id='load_aggregated_results',
    python_callable=load_aggregated_results,
    provide_context=True,
    dag=dag
)

# Task 5: Data quality checks
def data_quality_checks(**context):
    """Validate ETL results"""
    hook = MongoHook(conn_id='mongodb_production')
    client = hook.get_conn()
    db = client['production']

    # Get counts from XCom
    ti = context['task_instance']
    source_count = ti.xcom_pull(task_ids='validate_source_data', key='source_count')
    staging_count = ti.xcom_pull(task_ids='extract_to_staging', key='staging_count')
    aggregated_count = ti.xcom_pull(task_ids='load_aggregated_results', key='aggregated_count')

    logging.info(f"Pipeline counts - Source: {source_count}, Staging: {staging_count}, Aggregated: {aggregated_count}")

    # Validate counts match (allowing for aggregation)
    if staging_count < source_count * 0.95:  # Allow 5% variance
        raise ValueError(f"Staging count ({staging_count}) significantly lower than source ({source_count})")

    # Check for duplicates in staging
    duplicate_pipeline = [
        {'$group': {'_id': '$transaction_id', 'count': {'$sum': 1}}},
        {'$match': {'count': {'$gt': 1}}},
        {'$count': 'duplicate_count'}
    ]

    duplicates = list(db.staging_transactions.aggregate(duplicate_pipeline))
    duplicate_count = duplicates[0]['duplicate_count'] if duplicates else 0

    if duplicate_count > 0:
        raise ValueError(f"Found {duplicate_count} duplicate transactions in staging")

    # Record metrics
    metrics = {
        'pipeline_date': context['execution_date'].date().isoformat(),
        'source_count': source_count,
        'staging_count': staging_count,
        'aggregated_count': aggregated_count,
        'duplicate_count': duplicate_count,
        'success': True,
        'timestamp': datetime.utcnow()
    }

    db.etl_metrics.insert_one(metrics)

    return metrics

quality_task = PythonOperator(
    task_id='data_quality_checks',
    python_callable=data_quality_checks,
    provide_context=True,
    dag=dag
)

# Task 6: Cleanup staging
def cleanup_staging(**context):
    """Remove staging data after successful pipeline"""
    hook = MongoHook(conn_id='mongodb_production')
    client = hook.get_conn()
    db = client['production']

    execution_date = context['execution_date']

    # Keep staging for 7 days for debugging
    cutoff_date = execution_date - timedelta(days=7)

    result = db.staging_transactions.delete_many({
        'processed_date': {'$lt': cutoff_date.date().isoformat()}
    })

    logging.info(f"Deleted {result.deleted_count} old staging records")

    return result.deleted_count

cleanup_task = PythonOperator(
    task_id='cleanup_staging',
    python_callable=cleanup_staging,
    provide_context=True,
    dag=dag
)

# Task dependencies
validate_task >> extract_task >> transform_task >> load_task >> quality_task >> cleanup_task
```

### Monitoring DAG avec Sensors

```python
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.sensors.python import PythonSensor

# Wait for upstream DAG
wait_for_upstream = ExternalTaskSensor(
    task_id='wait_for_data_ingestion',
    external_dag_id='mongodb_realtime_ingestion',
    external_task_id='finalize_ingestion',
    timeout=3600,
    poke_interval=300,
    dag=dag
)

# Custom sensor: Check MongoDB collection ready
def check_mongodb_ready(**context):
    hook = MongoHook(conn_id='mongodb_production')
    client = hook.get_conn()
    db = client['production']

    # Check if collection has recent data
    recent_count = db.transactions.count_documents({
        'transaction_date': {'$gte': datetime.utcnow() - timedelta(hours=1)}
    })

    return recent_count > 0

mongodb_sensor = PythonSensor(
    task_id='check_mongodb_ready',
    python_callable=check_mongodb_ready,
    timeout=1800,
    poke_interval=60,
    dag=dag
)

wait_for_upstream >> mongodb_sensor >> validate_task
```

---

## 🔄 DBT (Data Build Tool) avec MongoDB

### Architecture dbt + MongoDB

```
┌───────────────────────────────────────────────────────────────┐
│              dbt + MongoDB Architecture                       │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  [Source: MongoDB Raw Data]                                   │
│           ↓                                                   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              dbt Models                                 │  │
│  │                                                         │  │
│  │  • staging/ (raw data cleaning)                         │  │
│  │  • marts/ (business logic)                              │  │
│  │  • metrics/ (KPIs)                                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│           ↓                                                   │
│  [Target: MongoDB Analytics Collections]                      │
│                                                               │
│  Features:                                                    │
│  • SQL-like transformations                                   │
│  • DAG dependency management                                  │
│  • Testing & documentation                                    │
│  • Version control                                            │
└───────────────────────────────────────────────────────────────┘
```

### Configuration dbt

**profiles.yml**

```yaml
mongodb_project:
  outputs:
    dev:
      type: mongodb
      host: localhost
      port: 27017
      database: analytics_dev
      username: dbt_user
      password: "{{ env_var('DBT_MONGODB_PASSWORD') }}"
      auth_source: admin

    prod:
      type: mongodb
      host: mongodb-cluster.example.com
      port: 27017
      database: analytics
      username: dbt_prod
      password: "{{ env_var('DBT_MONGODB_PROD_PASSWORD') }}"
      auth_source: admin
      ssl: true

  target: dev
```

**dbt_project.yml**

```yaml
name: 'mongodb_analytics'
version: '1.0.0'
config-version: 2

profile: 'mongodb_project'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  mongodb_analytics:
    staging:
      +materialized: view
      +schema: staging

    marts:
      +materialized: table
      +schema: marts

      sales:
        +materialized: incremental
        +unique_key: transaction_date
        +on_schema_change: "append_new_columns"
```

### Models dbt

**staging/stg_transactions.sql**

```sql
-- Staging: Clean raw transactions
{{
  config(
    materialized='view'
  )
}}

with source as (
    select * from {{ source('production', 'transactions') }}
),

cleaned as (
    select
        transaction_id,
        transaction_date,
        customer_id,
        coalesce(total, 0) as total,
        coalesce(status, 'unknown') as status,
        items,
        created_at,
        updated_at
    from source
    where
        -- Data quality filters
        transaction_date is not null
        and customer_id is not null
        and total >= 0
)

select * from cleaned
```

**marts/sales/fct_daily_sales.sql**

```sql
-- Fact table: Daily sales aggregations
{{
  config(
    materialized='incremental',
    unique_key='date'
  )
}}

with transactions as (
    select * from {{ ref('stg_transactions') }}
    {% if is_incremental() %}
    where transaction_date > (select max(date) from {{ this }})
    {% endif %}
),

daily_aggregated as (
    select
        date(transaction_date) as date,
        count(*) as transaction_count,
        sum(total) as revenue,
        avg(total) as avg_order_value,
        count(distinct customer_id) as unique_customers
    from transactions
    where status = 'completed'
    group by date(transaction_date)
)

select * from daily_aggregated
```

**marts/sales/dim_customers.sql**

```sql
-- Dimension: Customer with lifetime metrics
{{
  config(
    materialized='table'
  )
}}

with customers as (
    select * from {{ source('production', 'customers') }}
),

transactions as (
    select * from {{ ref('stg_transactions') }}
),

customer_metrics as (
    select
        customer_id,
        count(*) as total_orders,
        sum(total) as lifetime_value,
        avg(total) as avg_order_value,
        min(transaction_date) as first_order_date,
        max(transaction_date) as last_order_date,
        datediff(current_date, max(transaction_date)) as days_since_last_order
    from transactions
    where status = 'completed'
    group by customer_id
),

enriched as (
    select
        c.customer_id,
        c.name,
        c.email,
        c.tier,
        cm.total_orders,
        cm.lifetime_value,
        cm.avg_order_value,
        cm.first_order_date,
        cm.last_order_date,
        cm.days_since_last_order,
        case
            when cm.days_since_last_order <= 30 then 'active'
            when cm.days_since_last_order <= 90 then 'at_risk'
            else 'churned'
        end as customer_status
    from customers c
    left join customer_metrics cm on c.customer_id = cm.customer_id
)

select * from enriched
```

### Tests dbt

**models/staging/stg_transactions.yml**

```yaml
version: 2

models:
  - name: stg_transactions
    description: "Cleaned transactions from production"
    columns:
      - name: transaction_id
        description: "Unique transaction identifier"
        tests:
          - unique
          - not_null

      - name: total
        description: "Transaction total amount"
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 1000000

      - name: status
        description: "Transaction status"
        tests:
          - accepted_values:
              values: ['pending', 'completed', 'cancelled', 'refunded']

      - name: transaction_date
        description: "Date of transaction"
        tests:
          - not_null
          - dbt_utils.recency:
              datepart: day
              interval: 7
```

### Exécution dbt

```bash
# Test connections
dbt debug

# Run all models
dbt run

# Run specific models
dbt run --select stg_transactions
dbt run --select marts.sales.*

# Run incremental models
dbt run --select fct_daily_sales

# Full refresh
dbt run --select fct_daily_sales --full-refresh

# Test data quality
dbt test

# Generate documentation
dbt docs generate
dbt docs serve

# Snapshot (slowly changing dimensions)
dbt snapshot
```

---

## 🏗️ Patterns ETL Avancés

### Pattern 1: Change Data Capture (CDC) Pipeline

```python
# cdc_pipeline.py
from pymongo import MongoClient
from kafka import KafkaProducer
import json

class CDCPipeline:
    def __init__(self, mongo_uri, kafka_brokers):
        self.mongo_client = MongoClient(mongo_uri)
        self.db = self.mongo_client['production']
        self.producer = KafkaProducer(
            bootstrap_servers=kafka_brokers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    def watch_collection(self, collection_name):
        """Watch MongoDB Change Stream and publish to Kafka"""
        collection = self.db[collection_name]

        pipeline = [
            {'$match': {'operationType': {'$in': ['insert', 'update', 'delete']}}}
        ]

        with collection.watch(pipeline, full_document='updateLookup') as stream:
            for change in stream:
                self.process_change(change, collection_name)

    def process_change(self, change, collection_name):
        """Process and publish change event"""
        event = {
            'collection': collection_name,
            'operation': change['operationType'],
            'document_id': str(change['documentKey']['_id']),
            'timestamp': change['clusterTime'].as_datetime().isoformat(),
            'data': change.get('fullDocument', {})
        }

        # Publish to Kafka
        topic = f'mongodb.{collection_name}.changes'
        self.producer.send(topic, value=event)

        print(f"Published change: {event['operation']} on {collection_name}")

# Usage
pipeline = CDCPipeline(
    mongo_uri='mongodb://localhost:27017',
    kafka_brokers=['kafka1:9092', 'kafka2:9092']
)

pipeline.watch_collection('transactions')
```

### Pattern 2: Incremental ETL with Watermark

```python
# incremental_etl.py
from datetime import datetime, timedelta
from pymongo import MongoClient
import pandas as pd

class IncrementalETL:
    def __init__(self, mongo_uri):
        self.client = MongoClient(mongo_uri)
        self.db = self.client['production']
        self.watermark_db = self.client['etl_metadata']

    def get_last_watermark(self, job_name):
        """Get last processed timestamp"""
        watermark = self.watermark_db.watermarks.find_one({'job_name': job_name})
        return watermark['last_processed'] if watermark else datetime(2000, 1, 1)

    def set_watermark(self, job_name, timestamp):
        """Update watermark"""
        self.watermark_db.watermarks.update_one(
            {'job_name': job_name},
            {'$set': {
                'job_name': job_name,
                'last_processed': timestamp,
                'updated_at': datetime.utcnow()
            }},
            upsert=True
        )

    def extract_incremental(self, collection_name, date_field='updated_at'):
        """Extract only new/updated records since last watermark"""
        job_name = f"extract_{collection_name}"
        last_watermark = self.get_last_watermark(job_name)
        current_time = datetime.utcnow()

        print(f"Extracting {collection_name} from {last_watermark} to {current_time}")

        # Query incremental data
        query = {date_field: {'$gt': last_watermark, '$lte': current_time}}
        cursor = self.db[collection_name].find(query)

        # Convert to DataFrame for processing
        data = list(cursor)
        df = pd.DataFrame(data)

        if not df.empty:
            # Process data
            df = self.transform_data(df)

            # Load to target
            self.load_data(df, f"{collection_name}_processed")

            # Update watermark
            self.set_watermark(job_name, current_time)

        return len(data)

    def transform_data(self, df):
        """Apply transformations"""
        # Data cleaning
        df = df.dropna(subset=['customer_id', 'total'])

        # Data enrichment
        df['processed_at'] = datetime.utcnow()

        return df

    def load_data(self, df, target_collection):
        """Load to target collection"""
        records = df.to_dict('records')

        if records:
            self.db[target_collection].insert_many(records)
            print(f"Loaded {len(records)} records to {target_collection}")

# Usage
etl = IncrementalETL('mongodb://localhost:27017')
etl.extract_incremental('transactions')
```

### Pattern 3: Slowly Changing Dimensions (SCD Type 2)

```python
# scd_type2.py
from pymongo import MongoClient
from datetime import datetime

class SCDType2Handler:
    def __init__(self, mongo_uri):
        self.client = MongoClient(mongo_uri)
        self.db = self.client['warehouse']

    def update_dimension(self, collection_name, key_field, new_record):
        """Handle SCD Type 2 updates"""
        collection = self.db[collection_name]

        # Find current active record
        current = collection.find_one({
            key_field: new_record[key_field],
            'is_current': True
        })

        if not current:
            # First insert
            new_record['is_current'] = True
            new_record['effective_date'] = datetime.utcnow()
            new_record['end_date'] = None
            new_record['version'] = 1
            collection.insert_one(new_record)
            print(f"Inserted new record for {key_field}={new_record[key_field]}")

        else:
            # Check if data changed
            changed = self.has_changed(current, new_record)

            if changed:
                # Close current record
                collection.update_one(
                    {'_id': current['_id']},
                    {'$set': {
                        'is_current': False,
                        'end_date': datetime.utcnow()
                    }}
                )

                # Insert new version
                new_record['is_current'] = True
                new_record['effective_date'] = datetime.utcnow()
                new_record['end_date'] = None
                new_record['version'] = current.get('version', 1) + 1
                collection.insert_one(new_record)
                print(f"Updated record for {key_field}={new_record[key_field]} (version {new_record['version']})")

            else:
                print(f"No changes detected for {key_field}={new_record[key_field]}")

    def has_changed(self, old_record, new_record):
        """Check if record has changed (excluding metadata fields)"""
        exclude_fields = {'_id', 'is_current', 'effective_date', 'end_date', 'version'}

        for key in new_record:
            if key not in exclude_fields:
                if new_record.get(key) != old_record.get(key):
                    return True

        return False

    def get_point_in_time(self, collection_name, key_field, key_value, date):
        """Query dimension at specific point in time"""
        collection = self.db[collection_name]

        return collection.find_one({
            key_field: key_value,
            'effective_date': {'$lte': date},
            '$or': [
                {'end_date': {'$gt': date}},
                {'end_date': None}
            ]
        })

# Usage
scd = SCDType2Handler('mongodb://localhost:27017')

# Update customer dimension
customer = {
    'customer_id': 'CUST-123',
    'name': 'John Doe',
    'tier': 'gold',
    'email': 'john@example.com'
}

scd.update_dimension('dim_customers', 'customer_id', customer)

# Query historical state
historical = scd.get_point_in_time(
    'dim_customers',
    'customer_id',
    'CUST-123',
    datetime(2024, 1, 1)
)
```

---

## 📊 Data Quality Framework

### Great Expectations avec MongoDB

```python
# ge_mongodb_suite.py
import great_expectations as gx
from pymongo import MongoClient
import pandas as pd

class MongoDBDataQuality:
    def __init__(self, mongo_uri):
        self.client = MongoClient(mongo_uri)
        self.context = gx.get_context()

    def create_expectation_suite(self, suite_name="mongodb_transactions"):
        """Create expectation suite for MongoDB data"""

        # Get data from MongoDB
        df = self.get_dataframe('production', 'transactions')

        # Create datasource
        datasource = self.context.sources.add_or_update_pandas(name="mongodb_source")
        data_asset = datasource.add_dataframe_asset(name="transactions")
        batch_request = data_asset.build_batch_request(dataframe=df)

        # Create validator
        validator = self.context.get_validator(
            batch_request=batch_request,
            expectation_suite_name=suite_name
        )

        # Define expectations
        validator.expect_column_values_to_not_be_null("transaction_id")
        validator.expect_column_values_to_be_unique("transaction_id")
        validator.expect_column_values_to_not_be_null("customer_id")
        validator.expect_column_values_to_not_be_null("total")
        validator.expect_column_values_to_be_between("total", min_value=0, max_value=1000000)
        validator.expect_column_values_to_be_in_set("status",
            value_set=["pending", "completed", "cancelled", "refunded"])
        validator.expect_column_values_to_not_be_null("transaction_date")

        # Custom expectation: recent data
        validator.expect_column_max_to_be_between(
            "transaction_date",
            min_value=pd.Timestamp.now() - pd.Timedelta(days=7),
            max_value=pd.Timestamp.now()
        )

        # Save suite
        validator.save_expectation_suite(discard_failed_expectations=False)

        return validator

    def get_dataframe(self, database, collection):
        """Convert MongoDB collection to DataFrame"""
        db = self.client[database]
        cursor = db[collection].find().limit(10000)  # Sample for validation
        return pd.DataFrame(list(cursor))

    def run_validation(self, suite_name="mongodb_transactions"):
        """Run validation and generate report"""
        df = self.get_dataframe('production', 'transactions')

        datasource = self.context.get_datasource("mongodb_source")
        data_asset = datasource.get_asset("transactions")
        batch_request = data_asset.build_batch_request(dataframe=df)

        checkpoint = self.context.add_or_update_checkpoint(
            name="mongodb_checkpoint",
            validations=[
                {
                    "batch_request": batch_request,
                    "expectation_suite_name": suite_name
                }
            ]
        )

        result = checkpoint.run()

        # Generate data docs
        self.context.build_data_docs()

        return result

# Usage
dq = MongoDBDataQuality('mongodb://localhost:27017')

# Create expectations
dq.create_expectation_suite()

# Run validation
result = dq.run_validation()

if result.success:
    print("✓ All data quality checks passed")
else:
    print("✗ Data quality issues detected")
```

### Custom Data Quality Checks

```python
# custom_dq_checks.py
from pymongo import MongoClient
from datetime import datetime, timedelta
import logging

class DataQualityChecker:
    def __init__(self, mongo_uri):
        self.client = MongoClient(mongo_uri)
        self.db = self.client['production']
        self.logger = logging.getLogger(__name__)

    def run_all_checks(self):
        """Run all data quality checks"""
        checks = [
            self.check_record_count,
            self.check_null_rates,
            self.check_duplicates,
            self.check_referential_integrity,
            self.check_data_freshness,
            self.check_value_distributions
        ]

        results = {}
        for check in checks:
            check_name = check.__name__
            try:
                result = check()
                results[check_name] = {'status': 'pass', 'result': result}
            except AssertionError as e:
                results[check_name] = {'status': 'fail', 'error': str(e)}
                self.logger.error(f"{check_name} failed: {e}")

        return results

    def check_record_count(self):
        """Check daily record count within expected range"""
        yesterday = datetime.utcnow() - timedelta(days=1)

        count = self.db.transactions.count_documents({
            'transaction_date': {
                '$gte': yesterday.replace(hour=0, minute=0, second=0),
                '$lt': yesterday.replace(hour=23, minute=59, second=59)
            }
        })

        # Expected range: 10K - 100K transactions per day
        assert 10000 <= count <= 100000, f"Record count {count} outside expected range"

        return count

    def check_null_rates(self):
        """Check null rates for critical fields"""
        pipeline = [
            {
                '$group': {
                    '_id': None,
                    'total_count': {'$sum': 1},
                    'null_customer_id': {
                        '$sum': {'$cond': [{'$eq': ['$customer_id', None]}, 1, 0]}
                    },
                    'null_total': {
                        '$sum': {'$cond': [{'$eq': ['$total', None]}, 1, 0]}
                    }
                }
            }
        ]

        result = list(self.db.transactions.aggregate(pipeline))[0]

        null_rate_customer = result['null_customer_id'] / result['total_count']
        null_rate_total = result['null_total'] / result['total_count']

        assert null_rate_customer < 0.01, f"customer_id null rate too high: {null_rate_customer:.2%}"
        assert null_rate_total < 0.01, f"total null rate too high: {null_rate_total:.2%}"

        return {
            'null_rate_customer_id': null_rate_customer,
            'null_rate_total': null_rate_total
        }

    def check_duplicates(self):
        """Check for duplicate transaction IDs"""
        pipeline = [
            {'$group': {'_id': '$transaction_id', 'count': {'$sum': 1}}},
            {'$match': {'count': {'$gt': 1}}},
            {'$count': 'duplicate_count'}
        ]

        result = list(self.db.transactions.aggregate(pipeline))
        duplicate_count = result[0]['duplicate_count'] if result else 0

        assert duplicate_count == 0, f"Found {duplicate_count} duplicate transaction IDs"

        return duplicate_count

    def check_referential_integrity(self):
        """Check referential integrity between collections"""
        # Find transactions with invalid customer_id
        pipeline = [
            {
                '$lookup': {
                    'from': 'customers',
                    'localField': 'customer_id',
                    'foreignField': 'customer_id',
                    'as': 'customer'
                }
            },
            {
                '$match': {'customer': {'$size': 0}}
            },
            {
                '$count': 'orphan_count'
            }
        ]

        result = list(self.db.transactions.aggregate(pipeline))
        orphan_count = result[0]['orphan_count'] if result else 0

        assert orphan_count == 0, f"Found {orphan_count} transactions with invalid customer_id"

        return orphan_count

    def check_data_freshness(self):
        """Check data freshness (most recent record)"""
        most_recent = self.db.transactions.find_one(
            sort=[('transaction_date', -1)]
        )

        if most_recent:
            age_hours = (datetime.utcnow() - most_recent['transaction_date']).total_seconds() / 3600
            assert age_hours < 24, f"Most recent data is {age_hours:.1f} hours old"
            return age_hours

        raise AssertionError("No data found in collection")

    def check_value_distributions(self):
        """Check value distributions for anomalies"""
        pipeline = [
            {
                '$group': {
                    '_id': None,
                    'avg_total': {'$avg': '$total'},
                    'stddev_total': {'$stdDev': '$total'},
                    'min_total': {'$min': '$total'},
                    'max_total': {'$max': '$total'}
                }
            }
        ]

        result = list(self.db.transactions.aggregate(pipeline))[0]

        # Check for anomalies
        assert result['min_total'] >= 0, f"Negative total detected: {result['min_total']}"
        assert result['max_total'] < 1000000, f"Suspiciously high total: {result['max_total']}"

        return result

# Usage
checker = DataQualityChecker('mongodb://localhost:27017')
results = checker.run_all_checks()

for check_name, result in results.items():
    if result['status'] == 'pass':
        print(f"✓ {check_name}: PASS")
    else:
        print(f"✗ {check_name}: FAIL - {result['error']}")
```

---

## 📊 Scénario Réel : E-commerce Data Platform (24 mois)

**Contexte**
- E-commerce B2C, 50M transactions/mois
- Sources : MongoDB (operational), MySQL (legacy), APIs externes
- Objectif : Unified data platform pour analytics et ML
- Stack : Airflow, Spark, dbt, MongoDB, S3, Redshift

### Architecture Complète

```
┌───────────────────────────────────────────────────────────────┐
│            Production Data Platform Architecture              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  [Data Sources]                                               │
│  • MongoDB (transactions, users, products)                    │
│  • MySQL (legacy orders, inventory)                           │
│  • APIs (payment providers, shipping)                         │
│       │                                                       │
│       ↓                                                       │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Ingestion Layer (Airflow)                 │   │
│  │  • CDC from MongoDB (Change Streams)                   │   │
│  │  • Batch from MySQL (daily)                            │   │
│  │  • API polling (hourly)                                │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   ↓                                           │
│  ┌────────────────────────────────────────────────────────┐   │
│  │           Raw Data Lake (S3)                           │   │
│  │  • Parquet format                                      │   │
│  │  • Partitioned by date                                 │   │
│  │  • 7 days retention                                    │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   ↓                                           │
│  ┌────────────────────────────────────────────────────────┐   │
│  │       Transformation Layer (Spark + dbt)               │   │
│  │  • Spark: Heavy transformations (join, agg)            │   │
│  │  • dbt: Business logic (SQL)                           │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   ↓                                           │
│  ┌────────────────────────────────────────────────────────┐   │
│  │          Analytics Warehouse (Redshift)                │   │
│  │  • Star schema                                         │   │
│  │  • Aggregated tables                                   │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   ↓                                           │
│  ┌────────────────────────────────────────────────────────┐   │
│  │        Serving Layer (MongoDB + Redis)                 │   │
│  │  • MongoDB: Operational analytics                      │   │
│  │  • Redis: Real-time metrics cache                      │   │
│  └────────────────────────────────────────────────────────┘   │
│                   ↓                                           │
│  [Consumption]                                                │
│  • BI Tools (Tableau, Looker)                                 │
│  • ML Models (SageMaker)                                      │
│  • APIs (internal, external)                                  │
└───────────────────────────────────────────────────────────────┘
```

### Pipelines Clés

**Pipeline 1 : Daily Batch ETL**
- Schedule : 2 AM daily
- Duration : 90 min
- Volume : 50M transactions/mois
- Steps :
  1. Extract (MongoDB, MySQL) → S3 (30 min)
  2. Transform (Spark) → Clean, join, agg (45 min)
  3. Load (Redshift, MongoDB) → Analytics tables (15 min)

**Pipeline 2 : Real-time CDC**
- Source : MongoDB Change Streams
- Target : Kafka → Spark Streaming → MongoDB Analytics
- Latency : < 5 seconds
- Volume : 2K events/sec (peak)

**Pipeline 3 : ML Feature Pipeline**
- Schedule : Daily
- Duration : 120 min
- Steps :
  1. Feature extraction (Spark) → 500+ features
  2. Feature store (MongoDB)
  3. Model training (SageMaker) → 10 models
  4. Predictions (batch) → MongoDB

### Métriques Production

**Performance**
- Daily batch : 50M records in 90 min
- Real-time CDC : < 5s latency
- Data freshness : < 15 min
- SLA : 99.5% availability

**Volume**
- Raw data : 2 TB/month
- Processed : 500 GB/month
- Total storage : 50 TB (18 months history)

**Quality**
- Data quality checks : 50+ tests
- Success rate : 99.8%
- Alert response time : < 10 min

### Challenges et Solutions

**Challenge 1 : CDC lag spikes**
- **Symptôme** : Change Streams lag monte à 2 min
- **Cause** : Bulk operations (nightly maintenance)
- **Solution** : Maintenance window + CDC catch-up logic

**Challenge 2 : Data quality issues**
- **Symptôme** : 2% de données invalides détectées post-processing
- **Cause** : Validations insuffisantes en ingestion
- **Solution** : Great Expectations en ingestion + monitoring

**Challenge 3 : Cross-team coordination**
- **Symptôme** : Breaking changes non communiquées
- **Cause** : Pas de contract testing
- **Solution** : Schema registry + versioning + CI/CD checks

---

## 🎯 Bonnes Pratiques

### 1. Idempotence

```python
# Assurer idempotence des pipelines
def idempotent_load(df, collection_name, key_field):
    """Load data idempotently (can re-run safely)"""

    # Upsert au lieu d'insert
    for record in df.to_dict('records'):
        db[collection_name].update_one(
            {key_field: record[key_field]},
            {'$set': record},
            upsert=True
        )
```

### 2. Logging et Observability

```python
import logging
import time
from functools import wraps

def log_execution(func):
    """Decorator pour logger execution time et erreurs"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        logger = logging.getLogger(func.__name__)

        try:
            logger.info(f"Starting {func.__name__}")
            result = func(*args, **kwargs)
            duration = time.time() - start
            logger.info(f"Completed {func.__name__} in {duration:.2f}s")
            return result
        except Exception as e:
            logger.error(f"Failed {func.__name__}: {e}")
            raise

    return wrapper

@log_execution
def extract_data():
    # Extract logic
    pass
```

### 3. Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half_open

    def call(self, func, *args, **kwargs):
        if self.state == 'open':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'half_open'
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_success(self):
        self.failure_count = 0
        self.state = 'closed'

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = 'open'
```

---

## 📚 Checklist ETL Production

**Design**
- [ ] Architecture documentée (diagrammes, data flow)
- [ ] SLAs définis (latency, freshness, availability)
- [ ] Error handling strategy
- [ ] Idempotence garantie
- [ ] Backfill strategy

**Implementation**
- [ ] Orchestration (Airflow DAGs configurés)
- [ ] Transformations testées (unit tests)
- [ ] Data quality checks (Great Expectations)
- [ ] Logging centralisé
- [ ] Alerting configuré

**Performance**
- [ ] Partitioning optimal
- [ ] Indexing approprié
- [ ] Batch size tuning
- [ ] Parallelization
- [ ] Resource limits

**Monitoring**
- [ ] Metrics collectées (duration, volume, errors)
- [ ] Dashboards créés
- [ ] Alerts configurées
- [ ] On-call rotation définie

**Documentation**
- [ ] README pipeline
- [ ] Runbooks incidents
- [ ] Data lineage documenté
- [ ] Schema documentation

---

## 🎓 Conclusion

Les pipelines ETL modernes avec MongoDB requièrent :

**Architecture**
- ✅ Orchestration robuste (Airflow)
- ✅ Transformations testées (dbt, Spark)
- ✅ Data quality (Great Expectations)
- ✅ Monitoring exhaustif

**Patterns clés**
- CDC pour real-time
- Incremental pour batch efficace
- SCD pour dimensions historiques
- Idempotence pour résilience

**Outils recommandés**
- Airflow : Orchestration batch
- dbt : Transformations SQL
- Spark : Processing à grande échelle
- Great Expectations : Data quality

L'investissement dans une plateforme ETL moderne génère un **ROI significatif** : données fiables, time-to-insight réduit, self-service analytics.

---

**Section finale du chapitre** : 19.10 Cas d'Usage Complets - Synthèse avec 3 cas d'usage end-to-end détaillés.

⏭️ [Coexistence avec des bases relationnelles](/19-migration-integration/10-coexistence-bases-relationnelles.md)
