🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.8 Intégration avec Apache Spark

## Introduction

Apache Spark est devenu le standard de facto pour le traitement de données à grande échelle. Son intégration avec MongoDB permet de construire des pipelines ETL complexes, d'exécuter des jobs analytics distribués et d'entraîner des modèles de machine learning sur des volumes massifs. Le MongoDB Spark Connector offre une intégration native avec des optimisations performance critiques (partitioning intelligent, predicate pushdown, bulk writes).

Cette section explore les architectures, configurations et patterns pour exploiter Spark + MongoDB à l'échelle enterprise dans des environnements de production.

---

## 🎯 Architecture et Cas d'Usage

### Vue d'ensemble

**MongoDB Spark Connector** : Bridge natif entre Spark et MongoDB
- Lecture parallèle avec partitioning intelligent
- Predicate pushdown (filtres côté MongoDB)
- Aggregation pushdown (pipelines côté MongoDB)
- Bulk writes optimisés
- Support Spark SQL, DataFrames, Datasets

### Architecture de Production

```
┌───────────────────────────────────────────────────────────────┐
│           Production Spark + MongoDB Architecture             │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              Apache Spark Cluster                        │ │
│  │  • 1 Master (Active + Standby)                           │ │
│  │  • 50 Workers (8 cores, 32 GB RAM each)                  │ │
│  │  • Total: 400 cores, 1.6 TB RAM                          │ │
│  │                                                          │ │
│  │  Components:                                             │ │
│  │  • Spark Core (RDD, DataFrames)                          │ │
│  │  • Spark SQL                                             │ │
│  │  • Spark Streaming (micro-batches)                       │ │
│  │  • MLlib (Machine Learning)                              │ │
│  └────────────────┬─────────────────────────────────────────┘ │
│                   │                                           │
│                   │ MongoDB Spark Connector 10.x              │
│                   │                                           │
│                   ↓                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         MongoDB Atlas Cluster (Analytics)                │ │
│  │                                                          │ │
│  │  Read Preference: secondaryPreferred                     │ │
│  │  Analytics Nodes: 3x dedicated secondaries               │ │
│  │  • Node 1: M200 (96 GB RAM, 32 cores)                    │ │
│  │  • Node 2: M200 (96 GB RAM, 32 cores)                    │ │
│  │  • Node 3: M200 (96 GB RAM, 32 cores)                    │ │
│  │                                                          │ │
│  │  Collections:                                            │ │
│  │  • transactions (10 TB, 50B docs)                        │ │
│  │  • customers (500 GB, 100M docs)                         │ │
│  │  • products (50 GB, 10M docs)                            │ │
│  │                                                          │ │
│  │  Indexes optimized for Spark queries                     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │           Data Flow Examples                             │ │
│  │                                                          │ │
│  │  1. Batch ETL:                                           │ │
│  │     MongoDB → Spark → Transform → S3/Parquet             │ │
│  │                                                          │ │
│  │  2. Analytics:                                           │ │
│  │     MongoDB → Spark SQL → Aggregations → Results         │ │
│  │                                                          │ │
│  │  3. ML Pipeline:                                         │ │
│  │     MongoDB → Feature Eng → MLlib Training → Model       │ │
│  │                                                          │ │
│  │  4. Data Enrichment:                                     │ │
│  │     MongoDB → Spark → Join External → Write Back         │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Cas d'usage principaux

**1. ETL à grande échelle**
- Migration data warehouse → Data lake
- Transformation complexe (dénormalisation, enrichissement)
- Agrégations lourdes

**2. Analytics distribués**
- Queries complexes sur volumes massifs (To+)
- Reporting batch (quotidien, hebdomadaire)
- Data science exploratoire

**3. Machine Learning**
- Feature engineering sur données brutes
- Training modèles distribués (MLlib)
- Batch prediction à grande échelle

**4. Real-time Streaming**
- Spark Streaming + MongoDB Change Streams
- Windowed aggregations
- Enrichissement temps réel

---

## 📦 MongoDB Spark Connector

### Installation et Setup

**Maven (POM)**
```xml
<dependency>
  <groupId>org.mongodb.spark</groupId>
  <artifactId>mongo-spark-connector_2.12</artifactId>
  <version>10.2.1</version>
</dependency>
```

**SBT**
```scala
libraryDependencies += "org.mongodb.spark" %% "mongo-spark-connector" % "10.2.1"
```

**spark-submit avec package**
```bash
spark-submit \
  --packages org.mongodb.spark:mongo-spark-connector_2.12:10.2.1 \
  --master spark://master:7077 \
  --deploy-mode cluster \
  --driver-memory 8g \
  --executor-memory 16g \
  --executor-cores 4 \
  --num-executors 50 \
  --conf "spark.mongodb.read.connection.uri=mongodb+srv://cluster.mongodb.net/production" \
  --conf "spark.mongodb.write.connection.uri=mongodb+srv://cluster.mongodb.net/output" \
  my-spark-job.jar
```

### Configuration de base

**SparkSession configuration**

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
  .appName("MongoDB Spark Integration")
  .master("spark://master:7077")

  // MongoDB read configuration
  .config("spark.mongodb.read.connection.uri",
    "mongodb+srv://user:password@cluster.mongodb.net/production")
  .config("spark.mongodb.read.database", "production")
  .config("spark.mongodb.read.collection", "transactions")

  // MongoDB write configuration
  .config("spark.mongodb.write.connection.uri",
    "mongodb+srv://user:password@cluster.mongodb.net/output")
  .config("spark.mongodb.write.database", "output")
  .config("spark.mongodb.write.collection", "results")

  // Read preferences
  .config("spark.mongodb.read.readPreference.name", "secondaryPreferred")
  .config("spark.mongodb.read.readPreference.tagSets",
    "[{ \"usage\": \"analytics\" }]")

  // Partitioning
  .config("spark.mongodb.read.partitioner",
    "com.mongodb.spark.sql.connector.read.partitioner.SamplePartitioner")
  .config("spark.mongodb.read.partitioner.samples.per.partition", "10")

  // Performance tuning
  .config("spark.mongodb.read.sampleSize", "10000")
  .config("spark.mongodb.write.maxBatchSize", "1024")

  .getOrCreate()
```

---

## 📖 Lecture de Données (Read Operations)

### Lecture basique

**Scala**
```scala
// Lecture collection complète
val df = spark.read
  .format("mongodb")
  .option("uri", "mongodb+srv://cluster.mongodb.net/production.transactions")
  .load()

df.show()
df.printSchema()

// Équivalent avec configuration globale
val df = spark.read
  .format("mongodb")
  .load()  // Utilise config par défaut
```

**PySpark**
```python
# Lecture collection complète
df = spark.read \
  .format("mongodb") \
  .option("uri", "mongodb+srv://cluster.mongodb.net/production.transactions") \
  .load()

df.show()
df.printSchema()
```

### Lecture avec filtres (Predicate Pushdown)

**Scala**
```scala
// Filtres poussés vers MongoDB (execution côté MongoDB)
val filteredDF = spark.read
  .format("mongodb")
  .load()
  .filter($"status" === "completed")
  .filter($"total" > 100)
  .filter($"transaction_date" >= "2024-01-01")

// Génère une query MongoDB:
// db.transactions.find({
//   status: "completed",
//   total: { $gt: 100 },
//   transaction_date: { $gte: ISODate("2024-01-01") }
// })
```

**PySpark**
```python
from pyspark.sql.functions import col

filtered_df = spark.read \
  .format("mongodb") \
  .load() \
  .filter(col("status") == "completed") \
  .filter(col("total") > 100) \
  .filter(col("transaction_date") >= "2024-01-01")
```

### Lecture avec pipeline d'agrégation

**Scala**
```scala
// Pipeline MongoDB exécuté côté MongoDB
val pipeline = """
[
  {
    "$match": {
      "transaction_date": { "$gte": { "$date": "2024-01-01T00:00:00Z" } }
    }
  },
  {
    "$group": {
      "_id": {
        "year": { "$year": "$transaction_date" },
        "month": { "$month": "$transaction_date" },
        "customer_tier": "$customer.tier"
      },
      "total_transactions": { "$sum": 1 },
      "total_revenue": { "$sum": "$total" },
      "avg_order_value": { "$avg": "$total" }
    }
  },
  {
    "$project": {
      "_id": 0,
      "year": "$_id.year",
      "month": "$_id.month",
      "customer_tier": "$_id.customer_tier",
      "total_transactions": 1,
      "total_revenue": 1,
      "avg_order_value": 1
    }
  }
]
"""

val aggregatedDF = spark.read
  .format("mongodb")
  .option("aggregation.pipeline", pipeline)
  .load()

aggregatedDF.show()

// Résultat est un DataFrame Spark, mais calculs exécutés dans MongoDB
```

**PySpark**
```python
pipeline = """
[
  { "$match": { "status": "completed" } },
  { "$group": {
      "_id": "$customer_tier",
      "total_revenue": { "$sum": "$total" }
    }
  }
]
"""

aggregated_df = spark.read \
  .format("mongodb") \
  .option("aggregation.pipeline", pipeline) \
  .load()
```

### Partitioning Strategies

**1. AutoBucketPartitioner (Default)**

```scala
// Partitioning automatique basé sur _id
val df = spark.read
  .format("mongodb")
  .option("partitioner", "com.mongodb.spark.sql.connector.read.partitioner.AutoBucketPartitioner")
  .option("partitioner.numberOfPartitions", "128")
  .load()

// Crée 128 partitions Spark basées sur ranges de _id
```

**2. SamplePartitioner**

```scala
// Sampling pour partitioning optimal (recommandé pour grandes collections)
val df = spark.read
  .format("mongodb")
  .option("partitioner", "com.mongodb.spark.sql.connector.read.partitioner.SamplePartitioner")
  .option("partitioner.samples.per.partition", "10")
  .load()

// Échantillonne collection pour déterminer partitions optimales
```

**3. ShardedPartitioner**

```scala
// Pour clusters MongoDB sharded (utilise shard keys)
val df = spark.read
  .format("mongodb")
  .option("partitioner", "com.mongodb.spark.sql.connector.read.partitioner.ShardedPartitioner")
  .load()

// Aligne partitions Spark avec shards MongoDB
```

**4. Custom Partitioner**

```scala
// Partitioner custom basé sur champ métier
val df = spark.read
  .format("mongodb")
  .option("partitioner", "com.mongodb.spark.sql.connector.read.partitioner.FieldPartitioner")
  .option("partitioner.field", "customer_tier")
  .load()

// Partitions: 1 par valeur unique de customer_tier
```

### Optimisations lecture

**Configuration avancée**

```scala
val optimizedDF = spark.read
  .format("mongodb")

  // Connection pooling
  .option("maxPoolSize", "100")
  .option("minPoolSize", "10")

  // Batch size pour cursors
  .option("batchSize", "10000")

  // Sample size pour schema inference
  .option("sampleSize", "10000")

  // Partitioning
  .option("partitioner", "SamplePartitioner")
  .option("partitioner.samples.per.partition", "10")

  // Read concern
  .option("readConcern.level", "majority")

  // SSL/TLS
  .option("ssl.enabled", "true")

  .load()
```

---

## 📝 Écriture de Données (Write Operations)

### Écriture basique

**Scala**
```scala
// DataFrame existant
val resultDF = spark.read
  .format("mongodb")
  .load()
  .groupBy("customer_tier")
  .agg(sum("total").as("total_revenue"))

// Écriture dans MongoDB
resultDF.write
  .format("mongodb")
  .mode("append")  // append, overwrite, ignore, error
  .option("database", "output")
  .option("collection", "revenue_by_tier")
  .save()
```

**PySpark**
```python
result_df = spark.read \
  .format("mongodb") \
  .load() \
  .groupBy("customer_tier") \
  .agg({"total": "sum"})

result_df.write \
  .format("mongodb") \
  .mode("append") \
  .option("database", "output") \
  .option("collection", "revenue_by_tier") \
  .save()
```

### Modes d'écriture

**1. Append (ajout)**
```scala
df.write
  .format("mongodb")
  .mode("append")
  .save()

// Insère tous documents (insertMany)
```

**2. Overwrite (remplacement complet)**
```scala
df.write
  .format("mongodb")
  .mode("overwrite")
  .save()

// 1. Drop collection
// 2. Insère tous documents
```

**3. Ignore (skip si collection existe)**
```scala
df.write
  .format("mongodb")
  .mode("ignore")
  .save()

// Ne fait rien si collection existe déjà
```

**4. Error (erreur si collection existe)**
```scala
df.write
  .format("mongodb")
  .mode("error")
  .save()

// Lève exception si collection existe
```

### Upsert (ReplaceDocument)

```scala
// Configuration pour upserts
df.write
  .format("mongodb")
  .mode("append")
  .option("replaceDocument", "true")
  .option("idFieldList", "customer_id,order_date")  // Champs pour matching
  .option("ordered", "false")  // Bulk writes non-ordonnés (performance)
  .save()

// Génère:
// db.collection.replaceOne(
//   { customer_id: x, order_date: y },
//   { document },
//   { upsert: true }
// )
```

### Bulk Writes Optimisés

```scala
// Configuration bulk writes
df.write
  .format("mongodb")
  .mode("append")

  // Batch size (documents par bulk operation)
  .option("maxBatchSize", "1024")

  // Ordered vs unordered
  .option("ordered", "false")  // Performance > Ordering

  // Write concern
  .option("writeConcern.w", "majority")
  .option("writeConcern.journal", "true")
  .option("writeConcern.wTimeoutMS", "5000")

  .save()
```

---

## 🔄 Spark SQL avec MongoDB

### Création de vues temporaires

```scala
// Lecture MongoDB
val transactionsDF = spark.read
  .format("mongodb")
  .option("database", "production")
  .option("collection", "transactions")
  .load()

// Créer vue SQL
transactionsDF.createOrReplaceTempView("transactions")

// Requêtes SQL
val results = spark.sql("""
  SELECT
    DATE(transaction_date) as date,
    customer_tier,
    COUNT(*) as transaction_count,
    SUM(total) as revenue,
    AVG(total) as avg_order_value
  FROM transactions
  WHERE
    transaction_date >= '2024-01-01'
    AND status = 'completed'
  GROUP BY DATE(transaction_date), customer_tier
  ORDER BY date DESC, revenue DESC
""")

results.show()

// Écriture résultats dans MongoDB
results.write
  .format("mongodb")
  .mode("overwrite")
  .option("collection", "daily_revenue_summary")
  .save()
```

### Joins complexes

```scala
// Lecture multiple collections
val transactions = spark.read
  .format("mongodb")
  .option("collection", "transactions")
  .load()
  .createOrReplaceTempView("transactions")

val customers = spark.read
  .format("mongodb")
  .option("collection", "customers")
  .load()
  .createOrReplaceTempView("customers")

val products = spark.read
  .format("mongodb")
  .option("collection", "products")
  .load()
  .createOrReplaceTempView("products")

// Join SQL
val enrichedTransactions = spark.sql("""
  SELECT
    t.transaction_id,
    t.transaction_date,
    c.customer_id,
    c.name as customer_name,
    c.tier as customer_tier,
    c.lifetime_value,
    t.items,
    t.total
  FROM transactions t
  JOIN customers c ON t.customer_id = c.customer_id
  WHERE t.transaction_date >= '2024-01-01'
""")

enrichedTransactions.write
  .format("mongodb")
  .mode("overwrite")
  .option("collection", "transactions_enriched")
  .save()
```

---

## 🌊 Spark Streaming avec MongoDB

### Architecture Streaming

```
┌────────────────────────────────────────────────────┐
│         Spark Streaming + MongoDB                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  [MongoDB Change Streams]                          │
│           ↓                                        │
│  [Kafka Topic: mongodb.changes]                    │
│           ↓                                        │
│  [Spark Structured Streaming]                      │
│           ↓                                        │
│  [Transformations (windowed aggregations)]         │
│           ↓                                        │
│  [MongoDB Sink (write results)]                    │
└────────────────────────────────────────────────────┘
```

### Structured Streaming

**Scala**
```scala
import org.apache.spark.sql.streaming._

// Read stream from Kafka (MongoDB Change Streams)
val changeStream = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "kafka1:9092,kafka2:9092")
  .option("subscribe", "mongodb.transactions.changes")
  .option("startingOffsets", "latest")
  .load()

// Parse JSON
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

val schema = new StructType()
  .add("_id", StringType)
  .add("operationType", StringType)
  .add("fullDocument", new StructType()
    .add("transaction_id", StringType)
    .add("customer_id", StringType)
    .add("total", DoubleType)
    .add("transaction_date", TimestampType)
  )

val parsedStream = changeStream
  .selectExpr("CAST(value AS STRING) as json")
  .select(from_json($"json", schema).as("data"))
  .select("data.*")
  .filter($"operationType" === "insert")
  .select(
    $"fullDocument.transaction_id",
    $"fullDocument.customer_id",
    $"fullDocument.total",
    $"fullDocument.transaction_date"
  )

// Windowed aggregation
val windowedAggregations = parsedStream
  .withWatermark("transaction_date", "10 minutes")
  .groupBy(
    window($"transaction_date", "5 minutes"),
    $"customer_id"
  )
  .agg(
    count("*").as("transaction_count"),
    sum("total").as("total_revenue"),
    avg("total").as("avg_order_value")
  )

// Write to MongoDB
val query = windowedAggregations.writeStream
  .format("mongodb")
  .option("checkpointLocation", "/tmp/checkpoint")
  .option("forceDeleteTempCheckpointLocation", "true")
  .outputMode("append")
  .option("database", "streaming_output")
  .option("collection", "windowed_aggregations")
  .start()

query.awaitTermination()
```

**PySpark**
```python
from pyspark.sql.functions import *
from pyspark.sql.types import *

# Read stream
change_stream = spark.readStream \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka1:9092") \
  .option("subscribe", "mongodb.transactions.changes") \
  .load()

# Parse and transform
schema = StructType([
  StructField("_id", StringType()),
  StructField("operationType", StringType()),
  StructField("fullDocument", StructType([
    StructField("transaction_id", StringType()),
    StructField("total", DoubleType()),
    StructField("transaction_date", TimestampType())
  ]))
])

parsed = change_stream \
  .selectExpr("CAST(value AS STRING) as json") \
  .select(from_json("json", schema).alias("data")) \
  .select("data.*")

# Windowed aggregation
windowed = parsed \
  .withWatermark("fullDocument.transaction_date", "10 minutes") \
  .groupBy(
    window("fullDocument.transaction_date", "5 minutes")
  ) \
  .agg(
    count("*").alias("count"),
    sum("fullDocument.total").alias("revenue")
  )

# Write to MongoDB
query = windowed.writeStream \
  .format("mongodb") \
  .option("checkpointLocation", "/tmp/checkpoint") \
  .outputMode("append") \
  .option("collection", "streaming_results") \
  .start()

query.awaitTermination()
```

---

## 🤖 Machine Learning avec MLlib

### Feature Engineering

```scala
import org.apache.spark.ml.feature._
import org.apache.spark.ml.Pipeline

// Lecture données brutes
val rawData = spark.read
  .format("mongodb")
  .option("collection", "transactions")
  .load()

// Feature engineering
val customerIndexer = new StringIndexer()
  .setInputCol("customer_tier")
  .setOutputCol("customer_tier_index")

val productIndexer = new StringIndexer()
  .setInputCol("product_category")
  .setOutputCol("product_category_index")

val assembler = new VectorAssembler()
  .setInputCols(Array(
    "total",
    "customer_tier_index",
    "product_category_index",
    "items_count"
  ))
  .setOutputCol("features")

val scaler = new StandardScaler()
  .setInputCol("features")
  .setOutputCol("scaled_features")

// Pipeline
val pipeline = new Pipeline()
  .setStages(Array(
    customerIndexer,
    productIndexer,
    assembler,
    scaler
  ))

val model = pipeline.fit(rawData)
val preparedData = model.transform(rawData)

// Sauvegarder features dans MongoDB
preparedData
  .select("transaction_id", "scaled_features")
  .write
  .format("mongodb")
  .mode("overwrite")
  .option("collection", "transaction_features")
  .save()
```

### Training Model (Classification)

```scala
import org.apache.spark.ml.classification.RandomForestClassifier
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator

// Lecture features
val features = spark.read
  .format("mongodb")
  .option("collection", "transaction_features")
  .load()

// Split train/test
val Array(trainingData, testData) = features.randomSplit(Array(0.8, 0.2), seed = 42)

// Random Forest Classifier
val rf = new RandomForestClassifier()
  .setLabelCol("label")
  .setFeaturesCol("scaled_features")
  .setNumTrees(100)
  .setMaxDepth(10)
  .setMaxBins(32)

val rfModel = rf.fit(trainingData)

// Prédictions
val predictions = rfModel.transform(testData)

// Évaluation
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("accuracy")

val accuracy = evaluator.evaluate(predictions)
println(s"Test Accuracy = $accuracy")

// Sauvegarder prédictions dans MongoDB
predictions
  .select("transaction_id", "prediction", "probability")
  .write
  .format("mongodb")
  .mode("overwrite")
  .option("collection", "predictions")
  .save()

// Sauvegarder modèle
rfModel.write.overwrite().save("/models/rf_model")
```

### Batch Prediction à grande échelle

```scala
// Charger modèle
import org.apache.spark.ml.classification.RandomForestClassificationModel

val loadedModel = RandomForestClassificationModel.load("/models/rf_model")

// Lecture nouvelles données (To+)
val newData = spark.read
  .format("mongodb")
  .option("collection", "transactions_new")
  .load()

// Préparation features (même pipeline)
val preparedNew = model.transform(newData)

// Prédictions en batch
val batchPredictions = loadedModel.transform(preparedNew)

// Écriture résultats (bulk write optimisé)
batchPredictions
  .select("transaction_id", "prediction", "probability")
  .write
  .format("mongodb")
  .mode("append")
  .option("maxBatchSize", "2048")
  .option("ordered", "false")
  .option("collection", "predictions_batch")
  .save()
```

---

## 🚀 Optimisations Performance

### 1. Partitioning Optimal

**Configuration recommandée**

```scala
// Pour collections < 100 GB
val df = spark.read
  .format("mongodb")
  .option("partitioner", "AutoBucketPartitioner")
  .option("partitioner.numberOfPartitions", "128")
  .load()

// Pour collections 100 GB - 1 TB
val df = spark.read
  .format("mongodb")
  .option("partitioner", "SamplePartitioner")
  .option("partitioner.samples.per.partition", "10")
  .load()

// Pour collections > 1 TB (sharded)
val df = spark.read
  .format("mongodb")
  .option("partitioner", "ShardedPartitioner")
  .load()
```

**Monitoring partitions**

```scala
println(s"Number of partitions: ${df.rdd.getNumPartitions}")

// Repartition si nécessaire
val repartitioned = df.repartition(256)

// Coalesce pour réduire (shuffle-less)
val coalesced = df.coalesce(64)
```

### 2. Predicate Pushdown

**Maximiser pushdown**

```scala
// GOOD: Filtres simples (pushdown complet)
val filtered = df
  .filter($"status" === "completed")
  .filter($"total" > 100)
  .filter($"date" >= "2024-01-01")

// GOOD: Projection (sélection champs)
val projected = df
  .select("transaction_id", "total", "date")
  .filter($"status" === "completed")

// BAD: UDF bloque pushdown
import org.apache.spark.sql.functions.udf
val customFilter = udf((x: String) => x.length > 5)
val noSpark = df.filter(customFilter($"status"))  // Pas de pushdown

// GOOD: Utiliser fonctions natives Spark
val goodPushdown = df.filter(length($"status") > 5)
```

### 3. Aggregation Pushdown

```scala
// Pipeline MongoDB exécuté côté MongoDB (optimal)
val pipeline = """
[
  { "$match": { "status": "completed" } },
  { "$group": {
      "_id": "$customer_tier",
      "total_revenue": { "$sum": "$total" },
      "count": { "$sum": 1 }
    }
  }
]
"""

val aggregated = spark.read
  .format("mongodb")
  .option("aggregation.pipeline", pipeline)
  .load()

// vs Spark-side aggregation (moins optimal)
val sparkAgg = spark.read
  .format("mongodb")
  .load()
  .filter($"status" === "completed")
  .groupBy("customer_tier")
  .agg(
    sum("total").as("total_revenue"),
    count("*").as("count")
  )
```

### 4. Write Performance

**Bulk writes optimisés**

```scala
df.write
  .format("mongodb")
  .mode("append")

  // Batch size élevé
  .option("maxBatchSize", "2048")

  // Unordered (parallèle)
  .option("ordered", "false")

  // Write concern optimisé
  .option("writeConcern.w", "1")  // Pas "majority" si pas critique
  .option("writeConcern.journal", "false")  // Trade-off durabilité/perf

  .save()

// Repartition avant write pour parallélisme
df.repartition(128)
  .write
  .format("mongodb")
  .save()
```

### 5. Memory Management

**Configuration Spark**

```scala
val spark = SparkSession.builder()
  .appName("MongoDB ETL")

  // Memory allocation
  .config("spark.executor.memory", "16g")
  .config("spark.driver.memory", "8g")
  .config("spark.memory.fraction", "0.8")
  .config("spark.memory.storageFraction", "0.3")

  // Shuffle
  .config("spark.sql.shuffle.partitions", "256")
  .config("spark.shuffle.compress", "true")
  .config("spark.shuffle.spill.compress", "true")

  // Serialization
  .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")

  // MongoDB connector tuning
  .config("spark.mongodb.keep_alive_ms", "60000")
  .config("spark.mongodb.maxConnectionIdleTime", "60000")

  .getOrCreate()
```

---

## 📊 Scénario Réel : Retail Analytics Pipeline (18 mois)

**Contexte**
- Retail chain, 1000 magasins
- MongoDB : 10 To transactions (50 milliards de documents)
- Objectif : Pipeline ETL quotidien + ML recommendations
- Spark cluster : 50 workers (400 cores, 1.6 TB RAM)

### Architecture Pipeline

```
┌────────────────────────────────────────────────────────────┐
│              Daily ETL & ML Pipeline                       │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  02:00 AM - ETL Job Start                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Step 1: Extract (MongoDB → Spark)                   │  │
│  │  • Read transactions (24h delta: ~150M docs)         │  │
│  │  • Read customers (full: 100M docs)                  │  │
│  │  • Read products (full: 10M docs)                    │  │
│  │  Duration: 15 min                                    │  │
│  └────────────────┬─────────────────────────────────────┘  │
│                   ↓                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Step 2: Transform (Spark processing)                │  │
│  │  • Join 3 collections (broadcast join)               │  │
│  │  • Aggregate by store/product/customer               │  │
│  │  • Calculate KPIs (inventory turnover, etc.)         │  │
│  │  • Feature engineering (ML)                          │  │
│  │  Duration: 45 min                                    │  │
│  └────────────────┬─────────────────────────────────────┘  │
│                   ↓                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Step 3: ML Training (MLlib)                         │  │
│  │  • Product recommendation model                      │  │
│  │  • Inventory forecast model                          │  │
│  │  • Customer churn prediction                         │  │
│  │  Duration: 30 min                                    │  │
│  └────────────────┬─────────────────────────────────────┘  │
│                   ↓                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Step 4: Load (Spark → MongoDB)                      │  │
│  │  • Write aggregated_daily (1M docs)                  │  │
│  │  • Write recommendations (10M docs)                  │  │
│  │  • Write forecasts (100K docs)                       │  │
│  │  Duration: 10 min                                    │  │
│  └────────────────┬─────────────────────────────────────┘  │
│                   ↓                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Step 5: Export (S3 Parquet)                         │  │
│  │  • Archive to Data Lake                              │  │
│  │  • Partitioned by date                               │  │
│  │  Duration: 20 min                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Total Duration: 2h (120 min)                              │
│  04:00 AM - Job Complete                                   │
└────────────────────────────────────────────────────────────┘
```

### Code Pipeline Complet

**Scala**
```scala
// retail-etl-pipeline.scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.ml.feature._
import org.apache.spark.ml.recommendation.ALS
import java.time.LocalDate

object RetailETLPipeline {

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("Retail Daily ETL Pipeline")
      .config("spark.mongodb.read.connection.uri",
        "mongodb+srv://cluster.mongodb.net/production")
      .config("spark.mongodb.write.connection.uri",
        "mongodb+srv://cluster.mongodb.net/analytics")
      .config("spark.mongodb.read.readPreference.name", "secondaryPreferred")
      .getOrCreate()

    import spark.implicits._

    val today = LocalDate.now()
    val yesterday = today.minusDays(1)

    println(s"Starting ETL pipeline for date: $yesterday")

    // Step 1: Extract
    println("Step 1: Extracting data...")

    val transactions = spark.read
      .format("mongodb")
      .option("collection", "transactions")
      .option("partitioner", "SamplePartitioner")
      .option("partitioner.samples.per.partition", "10")
      .load()
      .filter($"transaction_date" >= yesterday.toString)

    val customers = spark.read
      .format("mongodb")
      .option("collection", "customers")
      .load()

    val products = spark.read
      .format("mongodb")
      .option("collection", "products")
      .load()

    println(s"Transactions count: ${transactions.count()}")

    // Step 2: Transform
    println("Step 2: Transforming data...")

    // Join avec broadcast (customers et products sont petits)
    val enriched = transactions
      .join(broadcast(customers), "customer_id")
      .join(broadcast(products), transactions("items.product_id") === products("product_id"))

    // Aggregations
    val dailyAggregations = enriched
      .groupBy(
        $"transaction_date",
        $"store_id",
        $"product_id",
        $"customer_tier"
      )
      .agg(
        count("*").as("transaction_count"),
        sum("total").as("revenue"),
        avg("total").as("avg_order_value"),
        countDistinct("customer_id").as("unique_customers")
      )

    // KPIs
    val inventoryMetrics = enriched
      .groupBy("store_id", "product_id")
      .agg(
        sum("items.quantity").as("units_sold"),
        avg("inventory_level").as("avg_inventory"),
        (sum("items.quantity") / avg("inventory_level")).as("turnover_rate")
      )

    // Step 3: ML Training
    println("Step 3: Training ML models...")

    // Feature engineering pour recommendations
    val userProductInteractions = enriched
      .select(
        $"customer_id",
        $"product_id",
        $"items.quantity".as("rating")  // Proxy pour rating
      )
      .groupBy("customer_id", "product_id")
      .agg(sum("rating").as("total_purchases"))

    // ALS pour recommendations
    val als = new ALS()
      .setMaxIter(10)
      .setRegParam(0.1)
      .setUserCol("customer_id")
      .setItemCol("product_id")
      .setRatingCol("total_purchases")

    val alsModel = als.fit(userProductInteractions)

    // Générer recommendations (top 10 par user)
    val recommendations = alsModel.recommendForAllUsers(10)
      .select(
        $"customer_id",
        $"recommendations.product_id".as("recommended_products"),
        $"recommendations.rating".as("scores")
      )

    // Step 4: Load to MongoDB
    println("Step 4: Loading results to MongoDB...")

    dailyAggregations.write
      .format("mongodb")
      .mode("append")
      .option("collection", "daily_aggregations")
      .option("maxBatchSize", "2048")
      .option("ordered", "false")
      .save()

    inventoryMetrics.write
      .format("mongodb")
      .mode("overwrite")
      .option("collection", "inventory_metrics")
      .save()

    recommendations.write
      .format("mongodb")
      .mode("overwrite")
      .option("collection", "product_recommendations")
      .save()

    // Step 5: Export to S3 (Parquet)
    println("Step 5: Exporting to S3...")

    enriched
      .write
      .mode("append")
      .partitionBy("transaction_date", "store_id")
      .parquet(s"s3://data-lake/transactions/date=$yesterday/")

    // Cleanup
    spark.stop()

    println("ETL pipeline completed successfully")
  }
}
```

### Métriques de Production

**Performance Pipeline**
- Durée totale : 120 min (2h)
- Throughput lecture : 1.25M docs/sec
- Throughput écriture : 100K docs/sec
- Data processed : 150M transactions (500 GB)
- Data written : 11M docs (15 GB)

**Ressources Spark**
- Executors : 50
- Cores/executor : 8
- RAM/executor : 16 GB
- Total RAM utilisé : ~1 TB
- Shuffle writes : 200 GB

**MongoDB Performance**
- Read ops/sec : 50K (P95 sur analytics nodes)
- Write ops/sec : 15K (P95)
- Latency read : 5ms (P95)
- Latency write : 15ms (P95)
- Cache hit rate : 95%

### Optimisations Appliquées

**1. Broadcast Joins**
```scala
// Customers et products < 1 GB → broadcast
val enriched = transactions
  .join(broadcast(customers), "customer_id")
  .join(broadcast(products), "product_id")

// Économise shuffle massif
```

**2. Partitioning Stratégique**
```scala
// Repartition par store_id avant aggregations
val repartitioned = transactions
  .repartition($"store_id")
  .groupBy("store_id", "product_id")
  .agg(...)

// Réduit shuffle
```

**3. Caching Intelligent**
```scala
// Cache DataFrames réutilisés
customers.cache()
products.cache()

// Utilisation multiple sans re-lecture MongoDB
```

**4. Write Parallelism**
```scala
// Repartition avant write pour parallélisme optimal
dailyAggregations
  .repartition(128)
  .write
  .format("mongodb")
  .save()
```

### Challenges Rencontrés

**Challenge 1 : Memory OOM sur aggregations**
- **Symptôme** : Executors crashent avec OOM
- **Cause** : Shuffle partitions insuffisantes (200 par défaut)
- **Solution** : `spark.sql.shuffle.partitions = 512`

**Challenge 2 : Skew dans partitions**
- **Symptôme** : Certains tasks 10x plus lents
- **Cause** : Stores avec volumes inégaux
- **Solution** : Salting sur store_id + repartition

**Challenge 3 : Write throttling MongoDB**
- **Symptôme** : Write latency monte à 500ms
- **Cause** : Bulk writes trop gros
- **Solution** : `maxBatchSize = 1024` (au lieu de 2048)

---

## 🎯 Bonnes Pratiques

### 1. Sizing Spark Cluster

**Règles empiriques**
```
Cores nécessaires = Data Size (GB) / 2
RAM nécessaire = Data Size (GB) × 3

Exemple: 1 TB data
→ 500 cores (63 executors × 8 cores)
→ 3 TB RAM (63 executors × 48 GB)
```

### 2. Monitoring

```scala
// Logging custom metrics
import org.apache.spark.TaskContext

df.foreachPartition { partition =>
  val context = TaskContext.get()
  val partitionId = context.partitionId()
  val count = partition.size

  println(s"Partition $partitionId processed $count records")
}

// Accumulators pour compteurs
val errorCounter = spark.sparkContext.longAccumulator("Errors")

df.foreach { row =>
  try {
    // Process row
  } catch {
    case e: Exception =>
      errorCounter.add(1)
  }
}

println(s"Total errors: ${errorCounter.value}")
```

### 3. Error Handling

```scala
// Try-catch autour de jobs critiques
try {
  df.write
    .format("mongodb")
    .mode("append")
    .save()
} catch {
  case e: Exception =>
    println(s"Write failed: ${e.getMessage}")

    // Write to fallback (S3)
    df.write
      .mode("append")
      .parquet("s3://fallback/failed_writes/")

    // Alert ops
    sendAlert(s"MongoDB write failed: ${e.getMessage}")
}
```

---

## 📚 Checklist MongoDB + Spark

**Configuration**
- [ ] Spark Connector version compatible
- [ ] MongoDB URI avec credentials sécurisés
- [ ] Read preference configuré (secondaryPreferred)
- [ ] Partitioner optimal sélectionné
- [ ] Batch sizes ajustés

**Performance**
- [ ] Indexes MongoDB optimisés pour queries Spark
- [ ] Partitioning Spark aligné sur volume données
- [ ] Predicate pushdown maximisé
- [ ] Aggregation pushdown utilisé quand possible
- [ ] Broadcast joins pour petites tables

**Ressources**
- [ ] Sizing cluster approprié (cores, RAM)
- [ ] Memory fractions configurées
- [ ] Shuffle partitions ajustées
- [ ] Connection pooling optimisé

**Monitoring**
- [ ] Spark UI accessible
- [ ] Logs centralisés
- [ ] Métriques collectées (Prometheus)
- [ ] Alerting configuré

**Resilience**
- [ ] Checkpointing activé (streaming)
- [ ] Error handling robuste
- [ ] Fallback writes (S3)
- [ ] Retry logic

---

## 🎓 Conclusion

Spark + MongoDB = **Puissance analytics** at scale. Points clés :

**Quand utiliser ?**
- ✅ ETL complexe (To+)
- ✅ Analytics distribués
- ✅ Machine Learning à grande échelle
- ✅ Batch processing (jobs quotidiens/hebdomadaires)

**Performance**
- Throughput : 1M+ docs/sec (lecture)
- Scalability : Linéaire avec ajout nodes
- Latency : Minutes pour To+ datasets

**Alternatives**
- MongoDB Aggregation : Queries simples, < 100 GB
- BI Connector : Analytics SQL, < 500 GB
- Spark : ETL complexe, ML, > 500 GB

Spark est l'outil optimal pour traitement **batch à grande échelle** sur MongoDB. Investissement infrastructure conséquent mais performance et scalabilité exceptionnelles.

---

**Prochaine section** : 19.9 Validation et Tests de Migration - Stratégies de validation complète et tests automatisés pour migrations critiques.

⏭️ [ETL et Data Pipelines](/19-migration-integration/09-etl-data-pipelines.md)
