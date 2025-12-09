🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.3 MongoDB Relational Migrator

## Introduction

MongoDB Relational Migrator est l'outil officiel MongoDB conçu pour simplifier et accélérer la migration depuis les bases de données relationnelles (MySQL, PostgreSQL, Oracle, SQL Server) vers MongoDB. Lancé en 2022, il combine analyse intelligente du schéma, suggestions de modélisation assistées par ML, et génération automatique de code de migration.

Cette section présente une analyse approfondie de l'outil, de ses capacités avancées et de son utilisation optimale dans des contextes de production enterprise.

---

## 🎯 Positionnement et Philosophie

### Vision produit

**Objectif principal** : Réduire le temps et la complexité de migration relationnelle → MongoDB en automatisant :
1. L'analyse du schéma source
2. La recommandation de patterns de modélisation
3. La génération de code de transformation
4. La validation post-migration

**Différenciation vs autres outils**
- **Focus MongoDB natif** : Optimisé pour patterns MongoDB (pas de compromis multi-DB)
- **Intelligence de modélisation** : Suggestions ML-based selon patterns d'accès
- **Interface graphique** : Accessible aux équipes non-programmeurs
- **Gratuit** : Pas de licensing, inclus dans MongoDB Community

### Cas d'usage optimaux

| Scénario | Fit Score | Raison |
|----------|-----------|--------|
| Migration one-shot < 1 To | ⭐⭐⭐⭐⭐ | Sweet spot de l'outil |
| Schéma relationnel normalisé classique | ⭐⭐⭐⭐⭐ | Détection patterns automatique |
| Équipe débutante MongoDB | ⭐⭐⭐⭐⭐ | Apprentissage guidé |
| POC/MVP rapide | ⭐⭐⭐⭐⭐ | Setup en heures, pas jours |
| Migration continue/CDC | ⭐⭐ | Pas de support natif CDC |
| Transformations métier complexes | ⭐⭐⭐ | Limité, code custom requis |
| Multi-To avec transformations | ⭐⭐ | Performance limitée vs Spark |

---

## 🚀 Installation et Configuration

### Prérequis Système

**Matériel recommandé**
```
Minimum:
- CPU: 2 cores
- RAM: 4 Go
- Disk: 10 Go free

Recommandé (production):
- CPU: 8+ cores
- RAM: 16-32 Go
- Disk: SSD 100+ Go (pour staging des données)
- Network: 1 Gbps+ vers source et cible
```

**Logiciels**
```
- Java Runtime Environment (JRE) 11+
- JDBC Drivers pour source DB (fournis)
- MongoDB 4.4+ sur cible
```

### Installation Multi-Plateformes

**Linux**
```bash
# Download
wget https://downloads.mongodb.com/compass/mongodb-relational-migrator-1.4.1-linux-x64.tar.gz

# Extract
tar -xzf mongodb-relational-migrator-1.4.1-linux-x64.tar.gz

# Move to /opt
sudo mv mongodb-relational-migrator-1.4.1 /opt/relational-migrator

# Create symlink
sudo ln -s /opt/relational-migrator/bin/mongodb-relational-migrator /usr/local/bin/

# Launch
mongodb-relational-migrator
```

**macOS**
```bash
# Download
curl -O https://downloads.mongodb.com/compass/mongodb-relational-migrator-1.4.1-darwin-x64.dmg

# Install
open mongodb-relational-migrator-1.4.1-darwin-x64.dmg
# Drag to Applications

# Launch
open /Applications/MongoDB\ Relational\ Migrator.app
```

**Windows**
```powershell
# Download
Invoke-WebRequest -Uri "https://downloads.mongodb.com/compass/mongodb-relational-migrator-1.4.1-win32-x64.zip" -OutFile "relational-migrator.zip"

# Extract
Expand-Archive -Path relational-migrator.zip -DestinationPath "C:\Program Files\MongoDB\RelationalMigrator"

# Launch
& "C:\Program Files\MongoDB\RelationalMigrator\mongodb-relational-migrator.exe"
```

### Configuration Avancée

**Configuration fichier (config.json)**
```json
{
  "version": "1.4.1",
  "settings": {
    "logging": {
      "level": "INFO",
      "file": "/var/log/relational-migrator/app.log",
      "maxSize": "100MB",
      "maxFiles": 10
    },

    "performance": {
      "maxMemoryMB": 8192,
      "threadPoolSize": 8,
      "batchSize": 5000,
      "prefetchRows": 10000
    },

    "network": {
      "connectionTimeout": 30000,
      "socketTimeout": 60000,
      "maxRetries": 3,
      "retryDelayMs": 1000
    },

    "analysis": {
      "sampleSize": 1000,
      "maxDepth": 5,
      "detectArrayPatterns": true,
      "suggestEmbedding": true
    },

    "security": {
      "encryptConnectionStrings": true,
      "auditLog": true,
      "auditLogPath": "/var/log/relational-migrator/audit.log"
    }
  }
}
```

**Variables d'environnement**
```bash
# MongoDB connection
export MONGODB_URI="mongodb+srv://user:password@cluster.mongodb.net/mydb"

# Source database
export SOURCE_DB_HOST="postgres.example.com"
export SOURCE_DB_PORT="5432"
export SOURCE_DB_NAME="production"
export SOURCE_DB_USER="migrator"
export SOURCE_DB_PASSWORD="secure_password"

# Performance tuning
export MIGRATOR_MAX_MEMORY="8192m"
export MIGRATOR_THREADS="8"
export MIGRATOR_BATCH_SIZE="5000"

# Logging
export MIGRATOR_LOG_LEVEL="DEBUG"
export MIGRATOR_LOG_PATH="/var/log/relational-migrator"
```

---

## 📊 Phase 1 : Analyse du Schéma Source

### Connexion à la base source

**Interface graphique**
```
Source Database Configuration
├── Database Type: [PostgreSQL ▼]
├── Host: postgres.production.example.com
├── Port: 5432
├── Database: ecommerce_prod
├── Username: readonly_user
├── Password: ••••••••
├── Schema(s): public, sales, inventory
└── SSL Mode: [Require ▼]

Advanced Options:
├── Connection Pool Size: 10
├── Query Timeout: 60s
├── Fetch Size: 10000
└── Exclude Tables: audit_*, tmp_*, staging_*
```

**Configuration programmatique (projet.json)**
```json
{
  "projectName": "ecommerce-migration",
  "version": "1.0.0",
  "created": "2024-01-15T10:00:00Z",

  "source": {
    "type": "postgresql",
    "connection": {
      "host": "${SOURCE_DB_HOST}",
      "port": 5432,
      "database": "ecommerce_prod",
      "schema": ["public", "sales"],
      "username": "${SOURCE_DB_USER}",
      "password": "${SOURCE_DB_PASSWORD}",
      "ssl": true,
      "sslMode": "require",
      "sslRootCert": "/path/to/ca-cert.pem"
    },

    "options": {
      "fetchSize": 10000,
      "queryTimeout": 60,
      "connectionPoolSize": 10,

      "excludePatterns": [
        "audit_.*",
        "tmp_.*",
        ".*_backup",
        "staging_.*"
      ],

      "includeTables": [
        "public.customers",
        "public.orders",
        "public.order_items",
        "public.products",
        "public.categories",
        "sales.transactions"
      ],

      "excludeSystemTables": true,
      "excludeViews": true,
      "excludeMaterializedViews": false
    }
  },

  "target": {
    "type": "mongodb",
    "connection": {
      "uri": "${MONGODB_URI}",
      "database": "ecommerce",
      "authSource": "admin"
    },

    "options": {
      "writeConcern": {
        "w": "majority",
        "j": true,
        "wtimeout": 5000
      },
      "readConcern": "majority",
      "maxPoolSize": 50
    }
  }
}
```

### Analyse automatique

**Rapport d'analyse généré**

```json
{
  "analysis": {
    "timestamp": "2024-01-15T10:05:32Z",
    "duration_seconds": 312,
    "status": "completed",

    "summary": {
      "total_tables": 47,
      "total_columns": 823,
      "total_rows": 15234567,
      "total_size_gb": 123.45,
      "total_relationships": 89,
      "avg_relationship_depth": 3.2
    },

    "tables": [
      {
        "schema": "public",
        "table": "customers",
        "row_count": 1250000,
        "size_mb": 2340,
        "avg_row_size_bytes": 1872,
        "columns": 18,

        "columns_detail": [
          {
            "name": "id",
            "type": "integer",
            "nullable": false,
            "primary_key": true,
            "unique": true,
            "indexed": true
          },
          {
            "name": "email",
            "type": "varchar(255)",
            "nullable": false,
            "unique": true,
            "indexed": true,
            "cardinality": 1250000
          },
          {
            "name": "first_name",
            "type": "varchar(100)",
            "nullable": true,
            "max_length": 45,
            "avg_length": 12.3
          },
          {
            "name": "registration_date",
            "type": "timestamp",
            "nullable": false,
            "indexed": true,
            "min_value": "2015-01-01T00:00:00Z",
            "max_value": "2024-01-15T09:30:00Z"
          }
        ],

        "relationships": [
          {
            "type": "one_to_many",
            "target_table": "orders",
            "foreign_key": "customer_id",
            "cardinality": {
              "min": 0,
              "max": 523,
              "avg": 12.3,
              "median": 5,
              "percentile_95": 48
            },
            "recommendation": "reference",
            "confidence": 0.92,
            "reasoning": "High cardinality avg (12.3), potential unbounded growth"
          },
          {
            "type": "one_to_many",
            "target_table": "customer_addresses",
            "foreign_key": "customer_id",
            "cardinality": {
              "min": 0,
              "max": 5,
              "avg": 1.8,
              "median": 2,
              "percentile_95": 3
            },
            "recommendation": "embed",
            "confidence": 0.95,
            "reasoning": "Low cardinality avg (1.8), bounded growth (max 5), frequently accessed together"
          }
        ],

        "indexes": [
          {
            "name": "customers_pkey",
            "columns": ["id"],
            "type": "btree",
            "unique": true,
            "primary": true
          },
          {
            "name": "idx_customers_email",
            "columns": ["email"],
            "type": "btree",
            "unique": true
          },
          {
            "name": "idx_customers_registration",
            "columns": ["registration_date"],
            "type": "btree"
          }
        ],

        "constraints": [
          {
            "type": "check",
            "name": "email_format",
            "definition": "email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$'"
          },
          {
            "type": "check",
            "name": "valid_status",
            "definition": "status IN ('active', 'inactive', 'suspended')"
          }
        ],

        "data_quality": {
          "null_percentage": {
            "phone": 23.5,
            "middle_name": 67.8,
            "company": 45.2
          },
          "unique_percentage": {
            "email": 100.0,
            "phone": 98.7
          },
          "outliers_detected": [
            {
              "column": "age",
              "issue": "values outside [18, 120]",
              "count": 234
            }
          ]
        }
      },

      {
        "schema": "public",
        "table": "orders",
        "row_count": 8456000,
        "size_mb": 12340,

        "relationships": [
          {
            "type": "many_to_one",
            "target_table": "customers",
            "foreign_key": "customer_id",
            "recommendation": "denormalize_partial",
            "confidence": 0.88,
            "reasoning": "Frequently joined for customer info, consider embedding customer_name, email for queries"
          },
          {
            "type": "one_to_many",
            "target_table": "order_items",
            "foreign_key": "order_id",
            "cardinality": {
              "avg": 3.2,
              "max": 45,
              "percentile_95": 8
            },
            "recommendation": "embed",
            "confidence": 0.96,
            "reasoning": "Low avg cardinality (3.2), strong aggregation root, always accessed together"
          }
        ],

        "query_patterns": [
          {
            "pattern": "SELECT o.*, c.name, c.email FROM orders o JOIN customers c ON o.customer_id = c.id WHERE o.status = ?",
            "frequency": 45680,
            "avg_duration_ms": 23.4,
            "recommendation": "Embed customer snapshot in order document"
          },
          {
            "pattern": "SELECT COUNT(*), SUM(total) FROM orders WHERE customer_id = ? AND order_date >= ?",
            "frequency": 12340,
            "avg_duration_ms": 12.1,
            "recommendation": "Index on customer_id + order_date compound"
          }
        ]
      }
    ],

    "complexity_analysis": {
      "score": 67,
      "factors": {
        "table_count": 47,
        "relationship_depth": 5,
        "many_to_many_count": 12,
        "self_referencing_count": 3,
        "polymorphic_associations": 2
      },
      "estimated_migration_effort": "medium_to_high",
      "estimated_duration_days": "15-25"
    },

    "recommendations": [
      {
        "priority": "high",
        "category": "modeling",
        "message": "Consider embedding order_items into orders collection",
        "impact": "Reduces queries by ~40%, improves read performance",
        "effort": "low"
      },
      {
        "priority": "high",
        "category": "performance",
        "message": "Table 'product_images' contains BLOB fields avg 2.3 MB",
        "impact": "May exceed 16 MB document limit with multiple images",
        "recommendation": "Use GridFS or external storage (S3)",
        "effort": "medium"
      },
      {
        "priority": "medium",
        "category": "data_quality",
        "message": "23.5% NULL values in customers.phone",
        "recommendation": "Consider optional field or validation rules",
        "effort": "low"
      }
    ],

    "potential_issues": [
      {
        "severity": "high",
        "table": "product_reviews",
        "issue": "Unbounded array growth detected",
        "details": "Max 1234 reviews per product, avg growing",
        "recommendation": "Use bucketing pattern or separate collection",
        "auto_fixable": false
      },
      {
        "severity": "medium",
        "table": "order_history",
        "issue": "Self-referencing relationship (parent_order_id)",
        "details": "Recursive depth up to 7 levels",
        "recommendation": "Use MongoDB $graphLookup or materialized path",
        "auto_fixable": false
      }
    ]
  }
}
```

### Analyse des patterns d'accès

**Intégration avec logs PostgreSQL**

Relational Migrator peut analyser `pg_stat_statements` pour détecter les patterns de requêtes :

```sql
-- Activer pg_stat_statements sur PostgreSQL
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Export queries pour analyse
COPY (
  SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    rows
  FROM pg_stat_statements
  WHERE query NOT LIKE '%pg_stat_statements%'
  ORDER BY calls DESC
  LIMIT 1000
) TO '/tmp/query_patterns.csv' WITH CSV HEADER;
```

**Import dans Relational Migrator**
```
Analysis → Import Query Patterns → /tmp/query_patterns.csv
```

Relational Migrator utilise ces données pour :
- Identifier les jointures fréquentes → Suggestions d'embedding
- Détecter les queries analytics → Indexes recommandés
- Optimiser la modélisation selon les use cases réels

---

## 🎨 Phase 2 : Modélisation Assistée

### Interface de modélisation

**Vue ERD → Document Model**

```
┌────────────────────────────────────────────────────────────┐
│              Relational Migrator - Modeling View           │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  LEFT PANEL: SQL Schema (ERD)    RIGHT PANEL: MongoDB Model│
│  ┌──────────────────────────┐    ┌──────────────────────┐  │
│  │ [customers]              │    │ Collection: customers   │
│  │  • id (PK)               │───▶│ {                    │  │
│  │  • email                 │    │   _id: ObjectId      │  │
│  │  • name                  │    │   email: String      │  │
│  │  • phone                 │    │   name: String       │  │
│  │                          │    │   addresses: [       │  │
│  │ [customer_addresses]     │    │     {                │  │
│  │  • id (PK)               │──┐ │       type: String   │  │
│  │  • customer_id (FK) ─────┘ └▶ │       street: String │  │
│  │  • address_type          │    │       city: String   │  │
│  │  • street                │    │     }                │  │
│  │  • city                  │    │   ],                 │  │
│  │                          │    │   order_ids: [       │  │
│  │ [orders]                 │    │     ObjectId         │  │
│  │  • id (PK)               │───┐│   ]                  │  │
│  │  • customer_id (FK) ─────┘ └▶ │ }                    │  │
│  └──────────────────────────┘    └──────────────────────┘  │
│                                                            │
│  MIDDLE: Relationship Rules                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ customer_addresses → EMBED as "addresses" array      │  │
│  │   Cardinality: avg 1.8, max 5                        │  │
│  │   Confidence: 95%                                    │  │
│  │   [Accept] [Modify] [Reject]                         │  │
│  │                                                      │  │
│  │ orders → REFERENCE as "order_ids" array              │  │
│  │   Cardinality: avg 12.3, unbounded                   │  │
│  │   Confidence: 92%                                    │  │
│  │   [Accept] [Modify] [Reject]                         │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Règles de mapping personnalisées

**Configuration avancée des transformations**

```json
{
  "mappings": [
    {
      "source": {
        "schema": "public",
        "table": "customers"
      },

      "target": {
        "database": "ecommerce",
        "collection": "customers"
      },

      "field_mappings": [
        {
          "source_column": "id",
          "target_field": "legacy_id",
          "type": "int32",
          "indexed": true,
          "unique": true
        },
        {
          "source_column": "email",
          "target_field": "email",
          "type": "string",
          "transform": "lowercase",
          "validation": {
            "regex": "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
          },
          "indexed": true,
          "unique": true
        },
        {
          "source_columns": ["first_name", "last_name"],
          "target_field": "full_name",
          "type": "string",
          "transform": "concat",
          "separator": " ",
          "trim": true
        },
        {
          "source_column": "phone",
          "target_field": "phone",
          "type": "string",
          "transform": "normalize_phone",
          "options": {
            "country_code": "US",
            "format": "E164"
          },
          "nullable": true
        },
        {
          "source_column": "birth_date",
          "target_field": "birth_date",
          "type": "date",
          "transform": "date_format",
          "options": {
            "input_format": "yyyy-MM-dd",
            "output_format": "ISODate"
          }
        },
        {
          "source_column": "status",
          "target_field": "status",
          "type": "string",
          "transform": "map_values",
          "mapping": {
            "A": "active",
            "I": "inactive",
            "S": "suspended",
            "D": "deleted"
          },
          "default": "unknown"
        }
      ],

      "computed_fields": [
        {
          "name": "age",
          "type": "int32",
          "expression": "YEAR(CURRENT_DATE) - YEAR(birth_date)",
          "nullable": true
        },
        {
          "name": "customer_tier",
          "type": "string",
          "expression": "CASE WHEN total_spent > 10000 THEN 'platinum' WHEN total_spent > 5000 THEN 'gold' WHEN total_spent > 1000 THEN 'silver' ELSE 'bronze' END"
        },
        {
          "name": "migrated_at",
          "type": "date",
          "expression": "CURRENT_TIMESTAMP",
          "immutable": true
        }
      ],

      "embedded_documents": [
        {
          "source_table": "customer_addresses",
          "foreign_key": "customer_id",
          "target_field": "addresses",
          "as_array": true,

          "field_mappings": [
            {
              "source_column": "address_type",
              "target_field": "type"
            },
            {
              "source_column": "street",
              "target_field": "street"
            },
            {
              "source_column": "city",
              "target_field": "city"
            },
            {
              "source_column": "postal_code",
              "target_field": "postal_code"
            },
            {
              "source_column": "country",
              "target_field": "country",
              "default": "US"
            }
          ],

          "filters": [
            {
              "column": "is_active",
              "operator": "=",
              "value": true
            }
          ],

          "sort": [
            {
              "column": "is_primary",
              "direction": "DESC"
            },
            {
              "column": "created_at",
              "direction": "ASC"
            }
          ]
        }
      ],

      "references": [
        {
          "source_table": "orders",
          "foreign_key": "customer_id",
          "target_field": "order_ids",
          "as_array": true,
          "store_ids_only": true,
          "limit": null
        }
      ],

      "denormalized_fields": [
        {
          "source_table": "customer_stats",
          "foreign_key": "customer_id",
          "target_field": "stats",
          "as_object": true,

          "fields": [
            "total_orders",
            "total_spent",
            "avg_order_value",
            "last_order_date"
          ]
        }
      ],

      "validation_rules": {
        "json_schema": {
          "$jsonSchema": {
            "bsonType": "object",
            "required": ["email", "full_name", "status"],
            "properties": {
              "email": {
                "bsonType": "string",
                "pattern": "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
              },
              "status": {
                "enum": ["active", "inactive", "suspended", "deleted"]
              },
              "addresses": {
                "bsonType": "array",
                "maxItems": 10,
                "items": {
                  "bsonType": "object",
                  "required": ["type", "city", "country"],
                  "properties": {
                    "type": {
                      "enum": ["billing", "shipping", "primary"]
                    }
                  }
                }
              }
            }
          }
        }
      },

      "indexes": [
        {
          "keys": { "email": 1 },
          "options": { "unique": true }
        },
        {
          "keys": { "legacy_id": 1 },
          "options": { "unique": true }
        },
        {
          "keys": { "status": 1, "customer_tier": 1 }
        },
        {
          "keys": { "full_name": "text" }
        },
        {
          "keys": { "addresses.city": 1, "addresses.country": 1 }
        }
      ]
    },

    {
      "source": {
        "schema": "public",
        "table": "orders"
      },

      "target": {
        "database": "ecommerce",
        "collection": "orders"
      },

      "field_mappings": [
        {
          "source_column": "id",
          "target_field": "legacy_id",
          "type": "int32"
        },
        {
          "source_column": "order_number",
          "target_field": "order_number",
          "type": "string",
          "indexed": true,
          "unique": true
        },
        {
          "source_column": "customer_id",
          "target_field": "customer.id",
          "type": "int32"
        },
        {
          "source_column": "order_date",
          "target_field": "order_date",
          "type": "date"
        },
        {
          "source_column": "status",
          "target_field": "status",
          "type": "string"
        },
        {
          "source_column": "total_amount",
          "target_field": "total",
          "type": "decimal",
          "precision": 2
        }
      ],

      "denormalized_fields": [
        {
          "source_table": "customers",
          "join_on": {
            "left": "customer_id",
            "right": "id"
          },
          "target_field": "customer",
          "as_object": true,
          "fields": {
            "id": "id",
            "name": "full_name",
            "email": "email",
            "tier": "customer_tier"
          }
        }
      ],

      "embedded_documents": [
        {
          "source_table": "order_items",
          "foreign_key": "order_id",
          "target_field": "items",
          "as_array": true,

          "field_mappings": [
            {
              "source_column": "product_id",
              "target_field": "product_id"
            },
            {
              "source_column": "quantity",
              "target_field": "quantity"
            },
            {
              "source_column": "unit_price",
              "target_field": "unit_price",
              "type": "decimal"
            }
          ],

          "denormalized_fields": [
            {
              "source_table": "products",
              "join_on": {
                "left": "product_id",
                "right": "id"
              },
              "fields": {
                "name": "product_name",
                "sku": "product_sku"
              }
            }
          ],

          "computed_fields": [
            {
              "name": "subtotal",
              "type": "decimal",
              "expression": "quantity * unit_price"
            }
          ]
        }
      ],

      "computed_fields": [
        {
          "name": "summary",
          "type": "object",
          "fields": {
            "item_count": "COUNT(order_items.id)",
            "subtotal": "SUM(order_items.quantity * order_items.unit_price)",
            "tax": "total_amount * 0.20",
            "shipping": "shipping_cost"
          }
        }
      ]
    }
  ]
}
```

### Patterns de modélisation avancés

**1. Bucketing Pattern (collections avec arrays volumineuses)**

```json
{
  "source": {
    "table": "product_reviews"
  },

  "target": {
    "collection": "product_review_buckets"
  },

  "bucketing": {
    "enabled": true,
    "bucket_by": "product_id",
    "bucket_size": 50,
    "sort_by": "created_at",
    "sort_direction": "ASC"
  },

  "field_mappings": [
    {
      "source_column": "product_id",
      "target_field": "product_id"
    },
    {
      "source_column": "id",
      "target_field": "reviews.review_id"
    },
    {
      "source_column": "user_id",
      "target_field": "reviews.user_id"
    },
    {
      "source_column": "rating",
      "target_field": "reviews.rating"
    },
    {
      "source_column": "comment",
      "target_field": "reviews.comment"
    }
  ],

  "bucket_metadata": {
    "bucket_number": "AUTO_INCREMENT",
    "review_count": "COUNT",
    "avg_rating": "AVG(rating)",
    "created_at": "MIN(created_at)",
    "updated_at": "MAX(created_at)"
  }
}
```

**Résultat MongoDB**
```javascript
// Bucket 1
{
  _id: ObjectId(),
  product_id: 12345,
  bucket_number: 1,
  review_count: 50,
  avg_rating: 4.2,
  created_at: ISODate("2023-01-01"),
  updated_at: ISODate("2023-03-15"),
  reviews: [
    { review_id: 1, user_id: 101, rating: 5, comment: "Great product!" },
    // ... 49 more reviews
  ]
}

// Bucket 2
{
  _id: ObjectId(),
  product_id: 12345,
  bucket_number: 2,
  review_count: 50,
  reviews: [ /* 50 reviews */ ]
}
```

**2. Schema Versioning Pattern**

```json
{
  "schema_versioning": {
    "enabled": true,
    "version_field": "schema_version",
    "current_version": "2.0",

    "version_mappings": {
      "1.0": {
        "applies_to": "legacy data before 2024-01-01",
        "transformations": [
          {
            "field": "address",
            "from": "string",
            "to": "object",
            "migration": "parse_address_string"
          }
        ]
      },
      "2.0": {
        "applies_to": "new data after 2024-01-01",
        "structure": "current"
      }
    }
  }
}
```

**3. Polymorphic Pattern (héritage)**

```json
{
  "source": {
    "table": "vehicles"
  },

  "target": {
    "collection": "vehicles"
  },

  "polymorphic": {
    "discriminator_field": "vehicle_type",
    "discriminator_mapping": {
      "car": {
        "additional_fields": ["num_doors", "trunk_capacity"],
        "validation": {
          "num_doors": { "bsonType": "int", "minimum": 2, "maximum": 5 }
        }
      },
      "truck": {
        "additional_fields": ["payload_capacity", "bed_length"],
        "validation": {
          "payload_capacity": { "bsonType": "int", "minimum": 1000 }
        }
      },
      "motorcycle": {
        "additional_fields": ["engine_cc", "has_sidecar"],
        "validation": {
          "engine_cc": { "bsonType": "int", "minimum": 50, "maximum": 2000 }
        }
      }
    }
  }
}
```

---

## ⚙️ Phase 3 : Génération et Exécution de la Migration

### Code généré

Relational Migrator génère du code dans plusieurs langages :

**Node.js (TypeScript)**
```typescript
// Generated by MongoDB Relational Migrator
// Project: ecommerce-migration
// Date: 2024-01-15T10:30:00Z

import { MongoClient, Db, Collection } from 'mongodb';
import { Client as PgClient } from 'pg';
import { Transform } from 'stream';
import pLimit from 'p-limit';

interface MigrationConfig {
  source: {
    host: string;
    port: number;
    database: string;
    user: string;
    password: string;
  };
  target: {
    uri: string;
    database: string;
  };
  options: {
    batchSize: number;
    parallelism: number;
    dryRun: boolean;
  };
}

interface MigrationStats {
  startTime: Date;
  endTime?: Date;
  tablesProcessed: number;
  totalRows: number;
  successfulRows: number;
  failedRows: number;
  errors: Array<{ table: string; row: any; error: string }>;
}

class CustomerMigrator {
  private pgClient: PgClient;
  private mongoClient: MongoClient;
  private db: Db;
  private collection: Collection;
  private stats: MigrationStats;
  private config: MigrationConfig;

  constructor(config: MigrationConfig) {
    this.config = config;
    this.stats = {
      startTime: new Date(),
      tablesProcessed: 0,
      totalRows: 0,
      successfulRows: 0,
      failedRows: 0,
      errors: []
    };
  }

  async connect(): Promise<void> {
    // PostgreSQL connection
    this.pgClient = new PgClient({
      host: this.config.source.host,
      port: this.config.source.port,
      database: this.config.source.database,
      user: this.config.source.user,
      password: this.config.source.password,
      ssl: true
    });
    await this.pgClient.connect();
    console.log('✓ Connected to PostgreSQL');

    // MongoDB connection
    this.mongoClient = new MongoClient(this.config.target.uri);
    await this.mongoClient.connect();
    this.db = this.mongoClient.db(this.config.target.database);
    this.collection = this.db.collection('customers');
    console.log('✓ Connected to MongoDB');
  }

  async migrateCustomers(): Promise<void> {
    console.log('Starting migration of customers...');

    const query = `
      SELECT
        c.id,
        c.email,
        c.first_name,
        c.last_name,
        c.phone,
        c.birth_date,
        c.status,
        c.created_at,

        -- Embedded addresses
        COALESCE(
          json_agg(
            json_build_object(
              'type', ca.address_type,
              'street', ca.street,
              'city', ca.city,
              'postal_code', ca.postal_code,
              'country', ca.country,
              'is_primary', ca.is_primary
            ) ORDER BY ca.is_primary DESC, ca.created_at ASC
          ) FILTER (WHERE ca.id IS NOT NULL),
          '[]'
        ) as addresses,

        -- Stats from customer_stats table
        cs.total_orders,
        cs.total_spent,
        cs.last_order_date

      FROM customers c
      LEFT JOIN customer_addresses ca ON c.id = ca.customer_id AND ca.is_active = true
      LEFT JOIN customer_stats cs ON c.id = cs.customer_id
      GROUP BY c.id, cs.total_orders, cs.total_spent, cs.last_order_date
      ORDER BY c.id
    `;

    const cursor = this.pgClient.query(query);
    const limit = pLimit(this.config.options.parallelism);
    const batch: any[] = [];

    for await (const row of cursor) {
      const document = this.transformCustomerRow(row);
      batch.push(document);

      if (batch.length >= this.config.options.batchSize) {
        await this.insertBatch(batch.splice(0));
      }
    }

    // Insert remaining documents
    if (batch.length > 0) {
      await this.insertBatch(batch);
    }

    // Create indexes
    await this.createIndexes();

    console.log('✓ Customer migration completed');
  }

  private transformCustomerRow(row: any): any {
    return {
      legacy_id: row.id,
      email: row.email.toLowerCase(),
      full_name: `${row.first_name} ${row.last_name}`.trim(),
      phone: this.normalizePhone(row.phone),
      birth_date: row.birth_date ? new Date(row.birth_date) : null,
      status: this.mapStatus(row.status),

      addresses: JSON.parse(row.addresses || '[]').map((addr: any) => ({
        type: addr.type,
        street: addr.street,
        city: addr.city,
        postal_code: addr.postal_code,
        country: addr.country || 'US',
        is_primary: addr.is_primary || false
      })),

      stats: {
        total_orders: row.total_orders || 0,
        total_spent: row.total_spent || 0,
        last_order_date: row.last_order_date ? new Date(row.last_order_date) : null
      },

      customer_tier: this.calculateTier(row.total_spent),

      metadata: {
        created_at: new Date(row.created_at),
        migrated_at: new Date(),
        migration_version: '1.0'
      }
    };
  }

  private normalizePhone(phone: string | null): string | null {
    if (!phone) return null;
    // Remove non-digits
    const digits = phone.replace(/\D/g, '');
    // Format as E164 (US)
    if (digits.length === 10) {
      return `+1${digits}`;
    }
    return digits.length > 0 ? `+${digits}` : null;
  }

  private mapStatus(status: string): string {
    const mapping: Record<string, string> = {
      'A': 'active',
      'I': 'inactive',
      'S': 'suspended',
      'D': 'deleted'
    };
    return mapping[status] || 'unknown';
  }

  private calculateTier(totalSpent: number): string {
    if (totalSpent > 10000) return 'platinum';
    if (totalSpent > 5000) return 'gold';
    if (totalSpent > 1000) return 'silver';
    return 'bronze';
  }

  private async insertBatch(documents: any[]): Promise<void> {
    try {
      if (this.config.options.dryRun) {
        console.log(`[DRY RUN] Would insert ${documents.length} documents`);
        this.stats.successfulRows += documents.length;
        return;
      }

      const result = await this.collection.insertMany(documents, {
        ordered: false,
        writeConcern: { w: 'majority', j: true }
      });

      this.stats.successfulRows += result.insertedCount;
      this.stats.totalRows += documents.length;

      console.log(`Inserted ${result.insertedCount} documents (total: ${this.stats.successfulRows})`);

    } catch (error: any) {
      // Handle bulk write errors
      if (error.code === 11000) {
        // Duplicate key errors
        const duplicates = error.writeErrors?.length || 0;
        this.stats.failedRows += duplicates;
        console.warn(`⚠ ${duplicates} duplicate key errors`);
      } else {
        this.stats.failedRows += documents.length;
        this.stats.errors.push({
          table: 'customers',
          row: documents[0],
          error: error.message
        });
        console.error(`✗ Batch insert failed: ${error.message}`);
      }
    }
  }

  private async createIndexes(): Promise<void> {
    console.log('Creating indexes...');

    await this.collection.createIndexes([
      { key: { legacy_id: 1 }, unique: true },
      { key: { email: 1 }, unique: true },
      { key: { status: 1, customer_tier: 1 } },
      { key: { full_name: 'text' } },
      { key: { 'addresses.city': 1, 'addresses.country': 1 } },
      { key: { 'metadata.created_at': -1 } }
    ]);

    console.log('✓ Indexes created');
  }

  async validate(): Promise<void> {
    console.log('Validating migration...');

    // Row count validation
    const pgCount = await this.pgClient.query('SELECT COUNT(*) FROM customers');
    const mongoCount = await this.collection.countDocuments();

    console.log(`PostgreSQL: ${pgCount.rows[0].count} rows`);
    console.log(`MongoDB: ${mongoCount} documents`);

    if (parseInt(pgCount.rows[0].count) === mongoCount) {
      console.log('✓ Row counts match');
    } else {
      console.error('✗ Row count mismatch!');
    }

    // Sample validation
    const sampleSize = 100;
    const samples = await this.pgClient.query(
      'SELECT id FROM customers ORDER BY RANDOM() LIMIT $1',
      [sampleSize]
    );

    let matchCount = 0;
    for (const sample of samples.rows) {
      const pgRow = await this.pgClient.query(
        'SELECT * FROM customers WHERE id = $1',
        [sample.id]
      );
      const mongoDoc = await this.collection.findOne({ legacy_id: sample.id });

      if (mongoDoc && this.compareRecords(pgRow.rows[0], mongoDoc)) {
        matchCount++;
      }
    }

    console.log(`Sample validation: ${matchCount}/${sampleSize} matches`);
    if (matchCount === sampleSize) {
      console.log('✓ Sample validation passed');
    } else {
      console.warn(`⚠ ${sampleSize - matchCount} mismatches detected`);
    }
  }

  private compareRecords(pgRow: any, mongoDoc: any): boolean {
    return (
      pgRow.id === mongoDoc.legacy_id &&
      pgRow.email.toLowerCase() === mongoDoc.email &&
      pgRow.status === mongoDoc.status
    );
  }

  async printStats(): Promise<void> {
    this.stats.endTime = new Date();
    const duration = (this.stats.endTime.getTime() - this.stats.startTime.getTime()) / 1000;

    console.log('\n=== Migration Statistics ===');
    console.log(`Duration: ${duration.toFixed(2)}s`);
    console.log(`Total rows: ${this.stats.totalRows}`);
    console.log(`Successful: ${this.stats.successfulRows}`);
    console.log(`Failed: ${this.stats.failedRows}`);
    console.log(`Throughput: ${(this.stats.successfulRows / duration).toFixed(2)} rows/sec`);

    if (this.stats.errors.length > 0) {
      console.log(`\nErrors (${this.stats.errors.length}):`);
      this.stats.errors.slice(0, 10).forEach(err => {
        console.log(`  - ${err.table}: ${err.error}`);
      });
    }
  }

  async close(): Promise<void> {
    await this.pgClient.end();
    await this.mongoClient.close();
    console.log('✓ Connections closed');
  }
}

// Main execution
async function main() {
  const config: MigrationConfig = {
    source: {
      host: process.env.SOURCE_DB_HOST || 'localhost',
      port: parseInt(process.env.SOURCE_DB_PORT || '5432'),
      database: process.env.SOURCE_DB_NAME || 'ecommerce',
      user: process.env.SOURCE_DB_USER || 'postgres',
      password: process.env.SOURCE_DB_PASSWORD || ''
    },
    target: {
      uri: process.env.MONGODB_URI || 'mongodb://localhost:27017',
      database: process.env.MONGODB_DATABASE || 'ecommerce'
    },
    options: {
      batchSize: parseInt(process.env.BATCH_SIZE || '1000'),
      parallelism: parseInt(process.env.PARALLELISM || '4'),
      dryRun: process.env.DRY_RUN === 'true'
    }
  };

  const migrator = new CustomerMigrator(config);

  try {
    await migrator.connect();
    await migrator.migrateCustomers();
    await migrator.validate();
    await migrator.printStats();
  } catch (error) {
    console.error('Migration failed:', error);
    process.exit(1);
  } finally {
    await migrator.close();
  }
}

main();
```

### Exécution et Monitoring

**Mode dry-run (simulation)**
```bash
export DRY_RUN=true
export SOURCE_DB_HOST=postgres.prod.example.com
export MONGODB_URI=mongodb+srv://cluster.mongodb.net/ecommerce

node dist/migrator.js
```

**Output dry-run**
```
✓ Connected to PostgreSQL
✓ Connected to MongoDB
Starting migration of customers...
[DRY RUN] Would insert 1000 documents
[DRY RUN] Would insert 1000 documents
[DRY RUN] Would insert 1000 documents
...
✓ Customer migration completed
✓ Indexes would be created
Validating migration...
[DRY RUN] Skipping validation

=== Migration Statistics ===
Duration: 45.23s
Total rows: 1250000
Successful: 1250000 (simulated)
Failed: 0
Throughput: 27631.58 rows/sec (simulated)
```

**Exécution production**
```bash
export DRY_RUN=false
export BATCH_SIZE=5000
export PARALLELISM=8

# With logging
node dist/migrator.js 2>&1 | tee migration-$(date +%Y%m%d-%H%M%S).log
```

**Monitoring en temps réel**
```bash
# Terminal 1: Run migration
node dist/migrator.js

# Terminal 2: Monitor MongoDB
watch -n 5 'mongosh --quiet --eval "
  db.getSiblingDB(\"ecommerce\").customers.countDocuments()
"'

# Terminal 3: Monitor PostgreSQL
watch -n 5 'psql -h postgres.prod.example.com -U user -d ecommerce -c "
  SELECT COUNT(*) FROM customers
"'
```

---

## 📈 Scénarios Réels Détaillés

### Scénario 1 : E-commerce (15 tables, 500 Go)

**Contexte**
- MySQL 8.0, schéma normalisé classique
- 15 tables principales : customers, orders, order_items, products, categories, inventory, etc.
- 500 Go de données, 50M de commandes
- Objectif : Migration week-end avec downtime 48h max

**Phase 1 : Analyse (Jour 1-2)**

```
Relational Migrator Analysis Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tables Analyzed: 15
Total Size: 523 Go
Estimated Migration Time: 18-24 hours
Complexity Score: 62/100 (Medium)

Key Findings:
├─ ✓ Standard e-commerce schema
├─ ⚠ order_items table: 180M rows, potential perf bottleneck
├─ ⚠ product_images: BLOB fields, avg 1.2 MB → OK for MongoDB
└─ ✓ Relationships: Mostly 1-to-many, good for embedding

Recommendations:
1. [HIGH] Embed order_items into orders (avg 3.1 items/order)
2. [HIGH] Embed product_variants into products
3. [MEDIUM] Denormalize customer name/email in orders for analytics
4. [LOW] Consider separate collection for product_reviews (unbounded growth)
```

**Phase 2 : Modélisation (Jour 3-5)**

**Modèle cible décidé**

```javascript
// Collection: customers
{
  _id: ObjectId(),
  legacy_id: 12345,
  email: "customer@example.com",
  full_name: "John Doe",
  addresses: [  // Embedded from customer_addresses
    { type: "billing", street: "...", city: "..." },
    { type: "shipping", street: "...", city: "..." }
  ],
  stats: {  // Denormalized from order aggregates
    total_orders: 45,
    lifetime_value: 3500.00,
    last_order_date: ISODate("2024-01-10")
  }
}

// Collection: products
{
  _id: ObjectId(),
  sku: "PROD-001",
  name: "MongoDB Handbook",
  category: {  // Denormalized from categories
    id: 5,
    name: "Books",
    path: "/Books/Tech"
  },
  variants: [  // Embedded from product_variants
    { size: "Paperback", price: 29.99, stock: 100 },
    { size: "Hardcover", price: 39.99, stock: 50 }
  ],
  images: [  // Embedded from product_images
    { url: "https://cdn.example.com/prod-001-1.jpg", primary: true },
    { url: "https://cdn.example.com/prod-001-2.jpg", primary: false }
  ]
}

// Collection: orders
{
  _id: ObjectId(),
  order_number: "ORD-2024-00123",
  order_date: ISODate("2024-01-15"),

  customer: {  // Denormalized snapshot
    id: 12345,
    name: "John Doe",
    email: "customer@example.com"
  },

  items: [  // Embedded from order_items
    {
      product_id: ObjectId(),
      product_name: "MongoDB Handbook",  // Snapshot
      sku: "PROD-001",
      variant: "Paperback",
      quantity: 2,
      unit_price: 29.99,
      subtotal: 59.98
    }
  ],

  shipping_address: { /* ... */ },
  billing_address: { /* ... */ },

  summary: {
    subtotal: 59.98,
    tax: 11.99,
    shipping: 5.00,
    total: 76.97
  },

  status: "completed",

  fulfillment: {
    shipped_at: ISODate("2024-01-16"),
    delivered_at: ISODate("2024-01-18"),
    tracking_number: "1Z999AA10123456784"
  }
}
```

**Phase 3 : Migration (Vendredi soir → Dimanche matin)**

**Timeline**

```
Vendredi 20:00 - Freeze des écritures application
Vendredi 20:15 - Backup MySQL complet
Vendredi 20:30 - Démarrage migration Relational Migrator
Vendredi 23:30 - customers migrés (3h, 5M records)
Samedi 02:30 - products migrés (6h, 2M records)
Samedi 10:30 - orders migrés (14h, 50M records) ← Goulot
Samedi 12:00 - Création indexes MongoDB (1.5h)
Samedi 13:30 - Validation automatisée (0.5h)
Samedi 14:00 - Tests fonctionnels manuels (2h)
Samedi 16:00 - Déploiement nouvelle version app
Samedi 17:00 - Tests smoke production
Samedi 18:00 - Réouverture application ✓
```

**Résultats**
- ✅ Durée totale : 22h (sous objectif 48h)
- ✅ Downtime : 22h
- ✅ Validation : 100% row counts match
- ✅ Performance post-migration : +40% sur queries principales
- ⚠️ 23 duplicates détectés (corruption source), nettoyés manuellement

**Optimisations appliquées**
```javascript
// Configuration Relational Migrator optimisée
{
  "performance": {
    "batchSize": 10000,  // Augmenté de 5000 par défaut
    "parallelism": 12,   // 8 cores dédiés
    "prefetchRows": 50000
  },

  // Désactiver indexes pendant insert
  "createIndexesAfter": true,

  // Write concern relâché temporairement
  "writeConcern": {
    "w": 1,  // Au lieu de majority
    "j": false
  }
}
```

---

### Scénario 2 : SaaS B2B multi-tenant (PostgreSQL, 80 Go)

**Contexte**
- PostgreSQL 14, application SaaS
- 25 tenants (clients), schéma partagé avec discriminateur tenant_id
- 80 Go total, croissance 10 Go/mois
- Objectif : Migrer vers MongoDB avec isolation tenant par database

**Stratégie : Une database MongoDB par tenant**

```
PostgreSQL (shared schema)          MongoDB (database per tenant)
┌─────────────────────────┐         ┌──────────────────────────┐
│ public.customers        │         │ tenant_acme/             │
│  • id                   │────────▶│   customers              │
│  • tenant_id (FK)       │         │   orders                 │
│  • name                 │         │   ...                    │
│                         │         └──────────────────────────┘
│ public.orders           │         ┌──────────────────────────┐
│  • id                   │────────▶│ tenant_globex/           │
│  • tenant_id (FK)       │         │   customers              │
│  • customer_id (FK)     │         │   orders                 │
└─────────────────────────┘         └──────────────────────────┘
```

**Configuration Relational Migrator**

```json
{
  "multi_tenant": {
    "enabled": true,
    "strategy": "database_per_tenant",
    "tenant_discriminator": "tenant_id",

    "tenants": [
      {
        "id": 1,
        "name": "acme",
        "database": "tenant_acme"
      },
      {
        "id": 2,
        "name": "globex",
        "database": "tenant_globex"
      }
    ],

    "parallel_tenants": 4,  // Migrer 4 tenants en parallèle

    "table_mappings": {
      "customers": {
        "filter": "tenant_id = ${tenant.id}",
        "exclude_columns": ["tenant_id"],  // Pas besoin dans MongoDB
        "collection": "customers"
      },
      "orders": {
        "filter": "tenant_id = ${tenant.id}",
        "exclude_columns": ["tenant_id"],
        "collection": "orders"
      }
    }
  }
}
```

**Exécution**

```bash
# Migration par tenant avec Relational Migrator CLI
mongodb-relational-migrator migrate \
  --project saas-migration.json \
  --mode multi-tenant \
  --parallel-tenants 4 \
  --log-per-tenant
```

**Résultats**

```
Tenant Migration Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total Tenants: 25
Duration: 4h 15min
Parallel Execution: 4 tenants at a time

Success: 25/25 tenants ✓

Per-Tenant Stats:
┌─────────────┬────────────┬──────────────┬──────────┐
│ Tenant      │ Rows       │ Duration     │ Status   │
├─────────────┼────────────┼──────────────┼──────────┤
│ acme        │ 234,567    │ 12 min       │ ✓        │
│ globex      │ 456,123    │ 18 min       │ ✓        │
│ initech     │ 123,456    │ 8 min        │ ✓        │
│ hooli       │ 789,012    │ 25 min       │ ✓        │
│ ...         │ ...        │ ...          │ ...      │
└─────────────┴────────────┴──────────────┴──────────┘

Post-Migration:
✓ Isolation: Each tenant in separate database
✓ Backups: Granular per tenant
✓ Scaling: Can shard large tenants individually
✓ GDPR: Easy tenant deletion (drop database)
```

**Avantages de l'approche**
- ✅ **Isolation forte** : Security, performance, compliance
- ✅ **Backup granulaire** : Restauration par tenant
- ✅ **Scaling flexible** : Sharding individuel des gros tenants
- ✅ **Customisation** : Schéma différent par tenant possible
- ✅ **Onboarding** : Nouveau tenant = nouvelle database

---

### Scénario 3 : Legacy System (Oracle, 200 tables, 2 To)

**Contexte**
- Oracle 11g, schéma complexe legacy (10+ ans)
- 200 tables, profondeur relations 6 niveaux
- 2 To de données
- Contraintes : Transformations business complexes, procédures stockées

**Approche : Relational Migrator + Custom Scripts**

Relational Migrator seul insuffisant pour ce scénario. Approche hybride :

1. **Relational Migrator** : Analyse + modélisation initiale
2. **Custom scripts** : Transformations business complexes
3. **Validation** : Relational Migrator

**Phase 1 : Analyse avec Relational Migrator**

```
Analysis Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Complexity: VERY HIGH (92/100)

Issues Detected:
├─ ✗ 47 self-referencing tables (hierarchies)
├─ ✗ 23 many-to-many with attributes
├─ ✗ 156 stored procedures (business logic)
├─ ✗ 89 triggers (audit, calculations)
├─ ⚠ 12 tables with CLOB/BLOB (avg 5 MB)
└─ ⚠ Circular dependencies detected (15 cycles)

Recommendation:
❌ Direct migration not feasible
✓ Use analysis for modeling, implement custom migration
✓ Incremental migration recommended (by domain)
```

**Phase 2 : Modélisation manuelle guidée**

Utiliser les suggestions de Relational Migrator comme point de départ, puis ajuster manuellement.

**Phase 3 : Migration custom avec validation**

```python
# custom_oracle_migration.py
import cx_Oracle
from pymongo import MongoClient
from bson import Decimal128
import json

class LegacyMigrator:
    def __init__(self):
        self.oracle = cx_Oracle.connect("user/pass@oracle-host/ORCL")
        self.mongo = MongoClient("mongodb://localhost:27017")
        self.db = self.mongo['legacy_system']

        # Load mapping config from Relational Migrator export
        with open('mapping_config.json') as f:
            self.mapping = json.load(f)

    def migrate_with_business_logic(self):
        """Migration avec transformations business"""

        # Exemple : Hiérarchie employés (self-referencing)
        cursor = self.oracle.cursor()
        cursor.execute("""
            SELECT
                employee_id,
                name,
                manager_id,
                LEVEL as hierarchy_level,
                SYS_CONNECT_BY_PATH(name, '/') as path
            FROM employees
            START WITH manager_id IS NULL
            CONNECT BY PRIOR employee_id = manager_id
        """)

        for row in cursor:
            # Appliquer logique métier des procédures stockées
            doc = self.transform_employee(row)
            self.db.employees.insert_one(doc)

    def transform_employee(self, row):
        """Logique business complexe (ex-procédure stockée)"""

        # Calculs complexes implémentés en Python
        salary_grade = self.calculate_salary_grade(row)
        bonus_eligible = self.check_bonus_eligibility(row)

        return {
            "_id": row[0],
            "name": row[1],
            "manager_id": row[2],
            "hierarchy_level": row[3],
            "path": row[4],
            "computed": {
                "salary_grade": salary_grade,
                "bonus_eligible": bonus_eligible
            }
        }
```

**Durée totale : 4 mois** (vs 1 semaine pour scénarios simples)
- Semaine 1-2 : Analyse
- Semaine 3-6 : Modélisation + dev scripts custom
- Semaine 7-10 : Tests + validation
- Semaine 11-14 : Migration incrémentale par domaine
- Semaine 15-16 : Stabilisation

---

## 🎯 Bonnes Pratiques et Optimisations

### 1. Performance tuning

**Configuration optimale selon volume**

```javascript
// < 100 Go (default settings OK)
{
  "batchSize": 5000,
  "parallelism": 4,
  "prefetchRows": 10000
}

// 100-500 Go
{
  "batchSize": 10000,
  "parallelism": 8,
  "prefetchRows": 50000,
  "createIndexesAfter": true,
  "writeConcern": { "w": 1, "j": false }  // Temporaire pendant migration
}

// > 500 Go
{
  "batchSize": 20000,
  "parallelism": 16,
  "prefetchRows": 100000,
  "createIndexesAfter": true,
  "writeConcern": { "w": 1, "j": false },
  "bulkWriteOrdered": false  // Performance boost
}
```

### 2. Gestion des erreurs

**Configuration robuste**

```json
{
  "error_handling": {
    "mode": "continue",  // vs "abort"
    "max_errors_per_table": 100,
    "log_errors": true,
    "error_log_path": "/var/log/relational-migrator/errors.log",

    "retry": {
      "enabled": true,
      "max_attempts": 3,
      "backoff_ms": 1000,
      "retriable_errors": [
        "NetworkTimeout",
        "ConnectionRefused",
        "TemporaryFailure"
      ]
    },

    "dead_letter_queue": {
      "enabled": true,
      "collection": "_migration_dlq",
      "include_context": true
    }
  }
}
```

### 3. Validation approfondie

**Script de validation post-migration**

```javascript
// validate_migration.js
const { MongoClient } = require('mongodb');
const { Client: PgClient } = require('pg');

async function comprehensiveValidation() {
  const pg = new PgClient(/* config */);
  const mongo = new MongoClient(/* config */);

  await pg.connect();
  await mongo.connect();
  const db = mongo.db('ecommerce');

  const report = {
    timestamp: new Date(),
    checks: []
  };

  // 1. Row counts
  const tables = ['customers', 'products', 'orders'];
  for (const table of tables) {
    const pgCount = await pg.query(`SELECT COUNT(*) FROM ${table}`);
    const mongoCount = await db.collection(table).countDocuments();

    report.checks.push({
      name: `row_count_${table}`,
      pg_count: parseInt(pgCount.rows[0].count),
      mongo_count: mongoCount,
      match: parseInt(pgCount.rows[0].count) === mongoCount
    });
  }

  // 2. Aggregate validation (business metrics)
  const pgRevenue = await pg.query(
    `SELECT SUM(total_amount) as revenue FROM orders WHERE status = 'completed'`
  );

  const mongoRevenue = await db.collection('orders').aggregate([
    { $match: { status: 'completed' } },
    { $group: { _id: null, revenue: { $sum: '$total' } } }
  ]).toArray();

  const pgRev = parseFloat(pgRevenue.rows[0].revenue);
  const mongoRev = mongoRevenue[0]?.revenue || 0;
  const revenueDiff = Math.abs(pgRev - mongoRev);

  report.checks.push({
    name: 'total_revenue',
    pg_value: pgRev,
    mongo_value: mongoRev,
    difference: revenueDiff,
    match: revenueDiff < 0.01  // Tolérance centimes
  });

  // 3. Sample deep comparison (100 random records)
  const sampleIds = await pg.query(
    `SELECT id FROM orders ORDER BY RANDOM() LIMIT 100`
  );

  let sampleMatches = 0;
  for (const row of sampleIds.rows) {
    const pgOrder = await pg.query(
      `SELECT * FROM orders WHERE id = $1`,
      [row.id]
    );

    const mongoOrder = await db.collection('orders').findOne({
      legacy_id: row.id
    });

    if (compareOrders(pgOrder.rows[0], mongoOrder)) {
      sampleMatches++;
    }
  }

  report.checks.push({
    name: 'sample_comparison',
    sample_size: 100,
    matches: sampleMatches,
    accuracy: (sampleMatches / 100) * 100
  });

  // Generate report
  console.log(JSON.stringify(report, null, 2));

  const allPassed = report.checks.every(c => c.match || c.accuracy >= 99);
  process.exit(allPassed ? 0 : 1);
}

comprehensiveValidation();
```

### 4. Rollback plan

**Configuration backup automatique**

```json
{
  "backup": {
    "before_migration": true,
    "backup_path": "/backups/migration-$(date +%Y%m%d)",

    "source_backup": {
      "tool": "pg_dump",
      "compression": "gzip",
      "parallel_jobs": 4
    },

    "target_backup": {
      "tool": "mongodump",
      "oplog": true  // Point-in-time recovery
    }
  },

  "rollback": {
    "enabled": true,
    "checkpoint_every_n_tables": 5,
    "allow_rollback_after_completion": true,
    "retention_days": 30
  }
}
```

---

## ⚖️ Comparaison avec Alternatives

### Relational Migrator vs Custom Scripts

| Critère | Relational Migrator | Custom Scripts |
|---------|-------------------|----------------|
| **Setup time** | Minutes | Days/weeks |
| **Learning curve** | Low (GUI) | High (code) |
| **Flexibility** | Medium | Total |
| **Performance** | Good (1-2 To/day) | Excellent (configurable) |
| **Transformations** | Limited | Unlimited |
| **Cost** | Free | Dev time |
| **Maintenance** | MongoDB updates | Self-maintained |
| **Best for** | Standard migrations | Complex custom logic |

### Relational Migrator vs Debezium

| Critère | Relational Migrator | Debezium |
|---------|-------------------|----------|
| **Use case** | One-shot migration | Continuous sync |
| **Complexity** | Low | High |
| **Zero downtime** | ❌ | ✅ |
| **CDC support** | ❌ | ✅ (native) |
| **Latency** | N/A (batch) | <1 sec (streaming) |
| **Infrastructure** | Standalone | Kafka cluster required |
| **Best for** | POC, simple migrations | Production zero-downtime |

---

## 🚨 Limitations et Workarounds

### Limitations connues

**1. Pas de support CDC**
- **Limitation** : Migration one-shot uniquement
- **Workaround** : Combiner avec Debezium pour sync continue post-migration initiale

**2. Transformations limitées**
- **Limitation** : Logique business complexe non supportée
- **Workaround** : Générer code, modifier, exécuter manuellement

**3. Performance sur très gros volumes (>5 To)**
- **Limitation** : Throughput limité vs Spark
- **Workaround** : Migration par batches, ou utiliser Spark pour initial load

**4. Pas de support multi-database source**
- **Limitation** : Une connexion source à la fois
- **Workaround** : Exécuter migrations en séquence ou parallèle manuel

**5. Support SGBD limité**
- **Limitation** : MySQL, PostgreSQL, Oracle, SQL Server uniquement
- **Workaround** : Export → CSV → mongoimport pour autres SGBD

---

## 📚 Ressources et Support

### Documentation officielle
- Relational Migrator Docs: https://www.mongodb.com/docs/relational-migrator/
- Video tutorials: MongoDB University
- Community forum: https://www.mongodb.com/community/forums/

### Support
- MongoDB Community (free): Forums, Stack Overflow
- MongoDB Support (paid): Ticket system, SLA garanties
- Professional Services: Migration consulting available

---

## 🎯 Checklist Utilisation Relational Migrator

**Avant de commencer**
- [ ] Version Relational Migrator 1.4+
- [ ] Accès en lecture à base source (readonly_user)
- [ ] Accès en écriture à MongoDB cible
- [ ] Backup complet base source
- [ ] Environnement test disponible
- [ ] Downtime planifié et communiqué

**Phase analyse**
- [ ] Connexion source validée
- [ ] Toutes les tables visibles
- [ ] Analyse terminée sans erreur
- [ ] Rapport lu et compris
- [ ] Issues identifiées (BLOB, unbounded arrays, etc.)

**Phase modélisation**
- [ ] Tous les mappings revus manuellement
- [ ] Embeddings/references décidés
- [ ] Transformations configurées
- [ ] Indexes MongoDB planifiés
- [ ] Validation rules définies
- [ ] Schema versioning si applicable

**Phase migration**
- [ ] Dry-run exécuté avec succès
- [ ] Performance acceptable (throughput)
- [ ] Code généré validé
- [ ] Variables d'environnement configurées
- [ ] Logs activés
- [ ] Monitoring en place

**Post-migration**
- [ ] Validation row counts (100%)
- [ ] Validation agrégats métier
- [ ] Sample comparison (>99%)
- [ ] Tests fonctionnels passés
- [ ] Performance vérifiée
- [ ] Rollback plan testé
- [ ] Documentation migration complétée

---

**Prochaine section** : 19.4 Stratégies de migration incrémentale - Techniques de migration progressive avec zero downtime.

⏭️ [Stratégies de migration incrémentale](/19-migration-integration/04-strategies-migration-incrementale.md)
