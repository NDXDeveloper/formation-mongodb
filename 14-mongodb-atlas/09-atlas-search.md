🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.9 Atlas Search

## Introduction

**Atlas Search** transforme MongoDB en moteur de recherche full-text puissant en intégrant **Apache Lucene** directement dans la plateforme. Plus besoin d'architecture complexe avec Elasticsearch ou Solr : la recherche avancée, l'autocomplete, le fuzzy matching, et les facettes sont natifs. Cette section guide les équipes dans l'implémentation de fonctionnalités de recherche production-ready avec Atlas Search.

### 🎯 Objectifs de cette Section

- Comprendre l'architecture Atlas Search (Lucene intégré)
- Créer et configurer des index de recherche
- Maîtriser les différents types de recherche (text, autocomplete, fuzzy, faceted)
- Implémenter des analyseurs personnalisés
- Optimiser les performances de recherche
- Intégrer Atlas Search dans vos applications

---

## 🏗️ Architecture Atlas Search

### Vue d'Ensemble

```
┌───────────────────────────────────────────────────────────────────────┐
│                    ATLAS SEARCH ARCHITECTURE                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   APPLICATION                                                         │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │  MongoDB Query:                                                  ││
│   │  db.products.aggregate([                                         ││
│   │    { $search: { text: { query: "laptop", path: "name" } } }      ││
│   │  ])                                                              ││
│   └────────────────────────────┬─────────────────────────────────────┘│
│                                │                                      │
│                                ▼                                      │
│   MONGODB CLUSTER (Primary + Secondaries)                             │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │  Regular MongoDB Data                                            ││
│   │  • Documents stored in BSON                                      ││
│   │  • Standard CRUD operations                                      ││
│   │  • Aggregation pipelines                                         ││
│   └────────────────────────────┬─────────────────────────────────────┘│
│                                │ Change Streams                       │
│                                ▼                                      │
│   ATLAS SEARCH NODES (Dedicated, Managed by Atlas)                    │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │  ┌─────────────────────────────────────────────────────────────┐ ││
│   │  │ LUCENE INDEXES                                              │ ││
│   │  │ • Inverted indexes for full-text search                     │ ││
│   │  │ • Tokenization, stemming, stop words                        │ ││
│   │  │ • N-gram indexes for autocomplete                           │ ││
│   │  │ • Facet indexes for aggregations                            │ ││
│   │  └─────────────────────────────────────────────────────────────┘ ││
│   │                                                                  ││
│   │  Synchronization:                                                ││
│   │  • Near real-time (< 10 seconds typical)                         ││
│   │  • Change streams capture data modifications                     ││
│   │  • Automatic index updates                                       ││
│   └──────────────────────────────────────────────────────────────────┘│
│                                                                       │
│   KEY BENEFITS:                                                       │
│   ✅ No separate search infrastructure (Elasticsearch, Solr)          │
│   ✅ Unified query language (MongoDB aggregation)                     │
│   ✅ Single connection string                                         │
│   ✅ Automatic sync with database                                     │
│   ✅ Integrated with Atlas (monitoring, security, backups)            │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Atlas Search vs Alternatives

```
┌────────────────────────────────────────────────────────────────────────┐
│           ATLAS SEARCH vs ELASTICSEARCH vs TEXT INDEXES                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  FEATURE              ATLAS SEARCH    ELASTICSEARCH    TEXT INDEXES    │
│  ───────────────────────────────────────────────────────────────────── │
│  Full-Text Search     ✅ Advanced      ✅ Advanced      ⚠️ Basic        │
│  Fuzzy Search         ✅ Yes           ✅ Yes           ❌ No           │
│  Autocomplete         ✅ Yes           ✅ Yes           ❌ No           │
│  Faceted Search       ✅ Yes           ✅ Yes           ❌ No           │
│  Synonyms             ✅ Yes           ✅ Yes           ❌ No           │
│  Language Analysis    ✅ 40+ langs     ✅ 50+ langs     ⚠️ Limited      │
│  Scoring/Relevance    ✅ Advanced      ✅ Advanced      ⚠️ Basic        │
│                                                                        │
│  Infrastructure       ✅ Integrated    ❌ Separate      ✅ Built-in     │
│  Data Sync            ✅ Automatic     ⚠️ Manual        ✅ Automatic    │
│  Query Language       ✅ MongoDB       ❌ DSL           ✅ MongoDB      │
│  Maintenance          ✅ Zero          ⚠️ High          ✅ Low          │
│  Cost                 💰 Moderate      💰💰 High        💰 Low          │
│                                                                        │
│  RECOMMENDATION:                                                       │
│  • Atlas Search: Best for most use cases (integrated, powerful)        │
│  • Elasticsearch: When you need absolute bleeding-edge features        │
│  • Text Indexes: Simple case-insensitive search only                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Concepts de Base

### Search Index

```javascript
// Un Search Index définit comment les données sont indexées pour la recherche

// EXEMPLE 1: Index Simple (Text Search)
{
  "name": "products_search",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "category": {
        "type": "string",
        "analyzer": "lucene.keyword"
      }
    }
  }
}

// EXEMPLE 2: Index avec Autocomplete
{
  "name": "products_autocomplete",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": {
        "type": "autocomplete",
        "analyzer": "lucene.standard",
        "tokenization": "edgeGram",
        "minGrams": 2,
        "maxGrams": 15
      }
    }
  }
}

// EXEMPLE 3: Index Dynamique (tous les champs)
{
  "name": "products_dynamic",
  "mappings": {
    "dynamic": true  // Index all fields automatically
  }
}
```

### Analyzers

Les **analyzers** transforment le texte pour l'indexation et la recherche.

```
┌───────────────────────────────────────────────────────────────────────┐
│                       ANALYZER PIPELINE                               │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  INPUT TEXT: "The Quick Brown Foxes are Running!"                     │
│                                                                       │
│  STEP 1: CHARACTER FILTERS                                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • Remove HTML tags                                               │ │
│  │ • Normalize characters (ñ → n, é → e)                            │ │
│  │ • Pattern replacement                                            │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│  Output: "The Quick Brown Foxes are Running!"                         │
│                                                                       │
│  STEP 2: TOKENIZATION                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • Split into tokens (words)                                      │ │
│  │ • Whitespace, punctuation-based                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│  Output: ["The", "Quick", "Brown", "Foxes", "are", "Running"]         │
│                                                                       │
│  STEP 3: TOKEN FILTERS                                                │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • Lowercase                                                      │ │
│  │ • Remove stop words (the, are, is...)                            │ │
│  │ • Stemming (running → run, foxes → fox)                          │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│  Output: ["quick", "brown", "fox", "run"]                             │
│                                                                       │
│  FINAL INDEXED TERMS: ["quick", "brown", "fox", "run"]                │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Built-in Analyzers

```javascript
// Atlas Search Built-in Analyzers

// 1. lucene.standard - General purpose
// • Tokenizes on whitespace and punctuation
// • Lowercase
// • No stemming
{
  "analyzer": "lucene.standard"
}
// Input: "The Quick-Brown Foxes"
// Tokens: ["the", "quick", "brown", "foxes"]

// 2. lucene.english - English language
// • Standard tokenization
// • English stop words removed
// • English stemming
{
  "analyzer": "lucene.english"
}
// Input: "The running foxes are quick"
// Tokens: ["run", "fox", "quick"]  // "the", "are" removed

// 3. lucene.keyword - Exact match
// • No tokenization
// • Whole field as single term
{
  "analyzer": "lucene.keyword"
}
// Input: "Product SKU-12345"
// Token: ["Product SKU-12345"]  // Single token

// 4. lucene.whitespace - Split on whitespace only
{
  "analyzer": "lucene.whitespace"
}
// Input: "Quick-Brown"
// Tokens: ["Quick-Brown"]  // Preserves punctuation

// 5. lucene.simple - Lowercase + non-letter split
{
  "analyzer": "lucene.simple"
}
// Input: "Quick-Brown-123"
// Tokens: ["quick", "brown"]  // Numbers removed

// 6. Language-specific analyzers
{
  "analyzer": "lucene.french"   // French
  "analyzer": "lucene.german"   // German
  "analyzer": "lucene.spanish"  // Spanish
  // 40+ languages supported
}
```

---

## 🔎 Types de Recherche

### 1. Text Search (Basic)

```javascript
// Basic text search
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop computer",
        path: "name"
      }
    }
  },
  {
    $project: {
      name: 1,
      description: 1,
      score: { $meta: "searchScore" }
    }
  },
  {
    $limit: 10
  }
])

// Multi-field search
db.products.aggregate([
  {
    $search: {
      text: {
        query: "gaming laptop",
        path: ["name", "description", "tags"]
      }
    }
  }
])

// Search with boost (field importance)
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop",
        path: [
          { value: "name", multi: "keywordAnalyzer" },
          { value: "description", multi: "textAnalyzer", score: { boost: { value: 0.5 } } }
        ]
      }
    }
  }
])
```

### 2. Autocomplete

```javascript
// Create autocomplete index first
// Via Atlas UI or API:
{
  "name": "autocomplete_index",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": {
        "type": "autocomplete",
        "tokenization": "edgeGram",
        "minGrams": 2,
        "maxGrams": 15,
        "foldDiacritics": true
      }
    }
  }
}

// Query with autocomplete
db.products.aggregate([
  {
    $search: {
      autocomplete: {
        query: "lap",  // User types "lap"
        path: "name",
        tokenOrder: "sequential"
      }
    }
  },
  {
    $limit: 10
  },
  {
    $project: {
      name: 1,
      score: { $meta: "searchScore" }
    }
  }
])

// Results:
// - "Laptop Computer"
// - "Laptop Stand"
// - "Gaming Laptop"
// But NOT "Overlap" (sequential order)
```

### 3. Fuzzy Search (Typo Tolerance)

```javascript
// Fuzzy search allows typos
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptap",  // Typo for "laptop"
        path: "name",
        fuzzy: {
          maxEdits: 2,        // Allow up to 2 character changes
          prefixLength: 1,    // First character must match
          maxExpansions: 50   // Limit expansion for performance
        }
      }
    }
  }
])

// Results will include:
// - "laptop" (1 edit: p→p, a→o)
// - "laptops" (1 edit: add 's')

// Practical fuzzy settings
const fuzzySettings = {
  // For short queries (3-5 chars)
  short: { maxEdits: 1, prefixLength: 2 },

  // For medium queries (6-10 chars)
  medium: { maxEdits: 1, prefixLength: 3 },

  // For long queries (11+ chars)
  long: { maxEdits: 2, prefixLength: 3 }
}
```

### 4. Faceted Search

```javascript
// Faceted search for filtering + aggregations
db.products.aggregate([
  {
    $searchMeta: {
      facet: {
        operator: {
          text: {
            query: "laptop",
            path: ["name", "description"]
          }
        },
        facets: {
          // Category facets
          categoryFacet: {
            type: "string",
            path: "category",
            numBuckets: 10
          },
          // Price range facets
          priceFacet: {
            type: "number",
            path: "price",
            boundaries: [0, 500, 1000, 2000, 5000],
            default: "other"
          },
          // Brand facets
          brandFacet: {
            type: "string",
            path: "brand"
          }
        }
      }
    }
  }
])

// Result structure:
{
  "count": { "lowerBound": 1247 },
  "facet": {
    "categoryFacet": {
      "buckets": [
        { "_id": "Electronics", "count": 856 },
        { "_id": "Computers", "count": 391 }
      ]
    },
    "priceFacet": {
      "buckets": [
        { "_id": 0, "count": 234 },     // $0-$500
        { "_id": 500, "count": 567 },   // $500-$1000
        { "_id": 1000, "count": 334 },  // $1000-$2000
        { "_id": 2000, "count": 112 }   // $2000-$5000
      ]
    },
    "brandFacet": {
      "buckets": [
        { "_id": "Dell", "count": 345 },
        { "_id": "HP", "count": 289 },
        { "_id": "Lenovo", "count": 234 }
      ]
    }
  }
}
```

### 5. Compound Search (Multiple Criteria)

```javascript
// Combine multiple search operators
db.products.aggregate([
  {
    $search: {
      compound: {
        // MUST conditions (all required)
        must: [
          {
            text: {
              query: "laptop",
              path: "name"
            }
          }
        ],
        // SHOULD conditions (boost score if matched)
        should: [
          {
            text: {
              query: "gaming",
              path: "tags",
              score: { boost: { value: 2.0 } }
            }
          }
        ],
        // MUST_NOT conditions (exclude)
        mustNot: [
          {
            text: {
              query: "refurbished",
              path: "condition"
            }
          }
        ],
        // FILTER conditions (no impact on score)
        filter: [
          {
            range: {
              path: "price",
              gte: 500,
              lte: 2000
            }
          },
          {
            text: {
              query: "in_stock",
              path: "availability"
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      price: 1,
      tags: 1,
      score: { $meta: "searchScore" }
    }
  }
])

// This query finds:
// - Products with "laptop" in name (MUST)
// - Boosts "gaming" laptops (SHOULD)
// - Excludes refurbished items (MUST_NOT)
// - Filters price $500-$2000 (FILTER)
// - Filters in-stock only (FILTER)
```

---

## ⚙️ Configuration d'Index

### Création via Atlas UI

```
1. Atlas Dashboard → Cluster → Search Tab
2. Click "Create Search Index"
3. Choose:
   - Visual Editor (GUI)
   - JSON Editor (advanced)
4. Select collection
5. Define mappings
6. Name index
7. Create (takes 2-5 minutes)
```

### Création via Atlas CLI

```bash
# Create search index via CLI
atlas clusters search indexes create \
  --clusterName production-cluster \
  --file search-index.json

# search-index.json
{
  "name": "products_search",
  "database": "shop",
  "collectionName": "products",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": { "type": "string" },
      "description": { "type": "string" },
      "price": { "type": "number" }
    }
  }
}

# List search indexes
atlas clusters search indexes list \
  --clusterName production-cluster

# Delete search index
atlas clusters search indexes delete \
  --indexId <INDEX_ID> \
  --clusterName production-cluster
```

### Configuration via Terraform

```hcl
# Terraform: Atlas Search Index
resource "mongodbatlas_search_index" "products" {
  project_id   = var.atlas_project_id
  cluster_name = "production-cluster"

  name               = "products_search"
  database           = "shop"
  collection_name    = "products"

  analyzer           = "lucene.standard"
  search_analyzer    = "lucene.standard"

  mappings_dynamic   = false

  mappings_fields = jsonencode({
    name = {
      type = "string"
      analyzer = "lucene.english"
    }
    description = {
      type = "string"
      analyzer = "lucene.english"
    }
    category = {
      type = "string"
      analyzer = "lucene.keyword"
    }
    price = {
      type = "number"
    }
    tags = {
      type = "string"
      analyzer = "lucene.standard"
    }
  })
}

# Autocomplete index
resource "mongodbatlas_search_index" "products_autocomplete" {
  project_id      = var.atlas_project_id
  cluster_name    = "production-cluster"
  name            = "products_autocomplete"
  database        = "shop"
  collection_name = "products"

  mappings_dynamic = false
  mappings_fields = jsonencode({
    name = {
      type = "autocomplete"
      tokenization = "edgeGram"
      minGrams = 2
      maxGrams = 15
    }
  })
}
```

---

## 🎯 Cas d'Usage Avancés

### E-commerce Search

```javascript
// Complete e-commerce search with filters, sorting, facets
async function searchProducts(queryText, filters = {}) {
  const pipeline = [
    {
      $search: {
        compound: {
          must: [
            {
              text: {
                query: queryText,
                path: ["name", "description"],
                fuzzy: { maxEdits: 1 }
              }
            }
          ],
          filter: []
        }
      }
    }
  ];

  // Add price filter
  if (filters.minPrice || filters.maxPrice) {
    pipeline[0].$search.compound.filter.push({
      range: {
        path: "price",
        gte: filters.minPrice || 0,
        lte: filters.maxPrice || 999999
      }
    });
  }

  // Add category filter
  if (filters.category) {
    pipeline[0].$search.compound.filter.push({
      text: {
        query: filters.category,
        path: "category"
      }
    });
  }

  // Add brand filter
  if (filters.brands && filters.brands.length > 0) {
    pipeline[0].$search.compound.should = filters.brands.map(brand => ({
      text: { query: brand, path: "brand" }
    }));
  }

  // Project results
  pipeline.push({
    $project: {
      name: 1,
      description: 1,
      price: 1,
      category: 1,
      brand: 1,
      image: 1,
      rating: 1,
      score: { $meta: "searchScore" }
    }
  });

  // Sort by relevance (score)
  pipeline.push({
    $sort: { score: -1 }
  });

  // Pagination
  pipeline.push(
    { $skip: filters.page * filters.pageSize },
    { $limit: filters.pageSize }
  );

  return await db.collection('products').aggregate(pipeline).toArray();
}

// Usage
const results = await searchProducts("gaming laptop", {
  minPrice: 500,
  maxPrice: 2000,
  category: "Electronics",
  brands: ["Dell", "HP"],
  page: 0,
  pageSize: 20
});
```

### Autocomplete with Highlighting

```javascript
// Autocomplete with result highlighting
db.products.aggregate([
  {
    $search: {
      autocomplete: {
        query: "lap",
        path: "name"
      },
      highlight: {
        path: "name"
      }
    }
  },
  {
    $project: {
      name: 1,
      highlights: { $meta: "searchHighlights" },
      score: { $meta: "searchScore" }
    }
  },
  {
    $limit: 10
  }
])

// Result with highlights:
{
  "_id": ObjectId("..."),
  "name": "Gaming Laptop Pro",
  "score": 4.5,
  "highlights": [
    {
      "path": "name",
      "texts": [
        { "value": "Gaming ", "type": "text" },
        { "value": "Lap", "type": "hit" },  // Highlighted
        { "value": "top Pro", "type": "text" }
      ],
      "score": 4.5
    }
  ]
}
```

### Geo-Search with Text

```javascript
// Combine geospatial and text search
db.stores.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "coffee shop",
              path: "name"
            }
          }
        ],
        filter: [
          {
            geoWithin: {
              path: "location",
              circle: {
                center: {
                  type: "Point",
                  coordinates: [-73.9857, 40.7484]  // NYC
                },
                radius: 5000  // 5km
              }
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      location: 1,
      distance: {
        $geoNear: {
          near: { type: "Point", coordinates: [-73.9857, 40.7484] },
          distanceField: "distance",
          spherical: true
        }
      }
    }
  }
])
```

---

## 🚀 Performance et Optimisation

### Index Size et Resource Planning

```
┌───────────────────────────────────────────────────────────────────────┐
│                    SEARCH INDEX SIZING                                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  RULE OF THUMB:                                                       │
│  Search Index Size ≈ 10-30% of collection size                        │
│                                                                       │
│  EXAMPLE:                                                             │
│  • Collection: 100 GB                                                 │
│  • Search Index: 10-30 GB                                             │
│                                                                       │
│  FACTORS AFFECTING SIZE:                                              │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ ↑ More fields indexed      → Larger index                        │ │
│  │ ↑ More text analysis       → Larger index                        │ │
│  │ ↑ Autocomplete indexes     → 2-3x larger                         │ │
│  │ ↓ Keyword-only fields      → Smaller index                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  CLUSTER TIER REQUIREMENTS:                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Index Size     Recommended Tier                                  │ │
│  │ ─────────────────────────────────────────────────────────────    │ │
│  │ < 1 GB         M10+                                              │ │
│  │ 1-10 GB        M20-M30                                           │ │
│  │ 10-50 GB       M40-M50                                           │ │
│  │ 50-200 GB      M60-M80                                           │ │
│  │ > 200 GB       M140+                                             │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Query Performance Tips

```javascript
// ❌ BAD: Dynamic mapping (indexes everything)
{
  "mappings": {
    "dynamic": true  // Indexes all fields automatically
  }
}
// Problems:
// - Large index size
// - Slower indexing
// - Unnecessary fields indexed

// ✅ GOOD: Explicit mapping (only what you need)
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": { "type": "string" },
      "description": { "type": "string" }
      // Only fields you actually search
    }
  }
}

// ❌ BAD: Searching too many fields
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop",
        path: ["name", "description", "reviews", "specs", "tags", "metadata"]
        // Too many paths → slower
      }
    }
  }
])

// ✅ GOOD: Search specific fields
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop",
        path: ["name", "description"]  // Most relevant fields only
      }
    }
  }
])

// ✅ GOOD: Use compound for complex queries
// Instead of multiple $search stages
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [...],
        should: [...],
        filter: [...]
      }
    }
  }
])
```

### Monitoring Search Performance

```javascript
// Add search metrics to aggregation
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop",
        path: "name"
      }
    }
  },
  {
    $project: {
      name: 1,
      score: { $meta: "searchScore" },
      // Add search metadata
      searchHighlights: { $meta: "searchHighlights" },
      searchScoreDetails: { $meta: "searchScoreDetails" }
    }
  }
])

// Monitor via Atlas UI:
// - Search index size
// - Query execution time
// - Index build time
// - Resource utilization
```

---

## 📋 Best Practices

### Index Design Checklist

```
┌────────────────────────────────────────────────────────────────────────┐
│                   SEARCH INDEX BEST PRACTICES                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  INDEX CONFIGURATION                                                   │
│  ☐ Use explicit mappings (dynamic: false) for production              │
│  ☐ Index only fields you actually search                              │
│  ☐ Choose appropriate analyzer per field                              │
│  ☐ Use keyword analyzer for exact match fields (SKU, category)        │
│  ☐ Consider separate indexes for different search types               │
│                                                                        │
│  QUERY OPTIMIZATION                                                    │
│  ☐ Limit $project to necessary fields                                 │
│  ☐ Use $limit to control result set size                              │
│  ☐ Implement pagination (skip + limit)                                │
│  ☐ Cache frequent queries at application level                        │
│  ☐ Use compound queries instead of multiple $search stages            │
│                                                                        │
│  AUTOCOMPLETE                                                          │
│  ☐ Create dedicated autocomplete index                                │
│  ☐ Set appropriate minGrams (2-3) and maxGrams (10-15)                │
│  ☐ Limit results to 10-20 suggestions                                 │
│  ☐ Debounce user input (300-500ms)                                    │
│                                                                        │
│  PERFORMANCE                                                           │
│  ☐ Monitor index build times                                          │
│  ☐ Monitor query latency (target < 100ms)                             │
│  ☐ Right-size cluster for index size                                  │
│  ☐ Test with production-scale data                                    │
│                                                                        │
│  RELEVANCE                                                             │
│  ☐ Use boost for important fields                                     │
│  ☐ Test relevance with real queries                                   │
│  ☐ Consider synonyms for improved matching                            │
│  ☐ Implement fuzzy search for typo tolerance                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Common Pitfalls

```
❌ SEARCH ANTI-PATTERNS:

1. Dynamic Mapping in Production
   • Indexes all fields automatically
   • Large index size, slow performance
   → Solution: Use explicit mappings

2. No Pagination
   • Returning all results (thousands)
   • High memory usage, slow response
   → Solution: Implement $skip and $limit

3. Wrong Analyzer
   • Using standard analyzer for exact matches
   • Using keyword analyzer for full-text
   → Solution: Match analyzer to use case

4. Over-Boosting
   • Boosting every field by 10x
   • Relevance becomes meaningless
   → Solution: Subtle boosts (1.5-3.0x)

5. No Index for Autocomplete
   • Using regular text index for autocomplete
   • Poor user experience
   → Solution: Create dedicated autocomplete index

6. Ignoring Index Build Time
   • Not monitoring sync lag
   • Outdated results shown to users
   → Solution: Monitor index status
```

---

## 🏁 Résumé

### Points Clés

1. **Architecture**
   - Apache Lucene intégré
   - Sync automatique via change streams
   - Dedicated search nodes (Atlas-managed)
   - Query via MongoDB aggregation

2. **Index Types**
   - Text: Full-text search
   - Autocomplete: Prefix matching
   - Keyword: Exact match
   - Number/Date: Range queries

3. **Search Types**
   - Text search (avec boost)
   - Autocomplete (edgeGram)
   - Fuzzy (typo tolerance)
   - Faceted (filters + aggregations)
   - Compound (complex queries)

4. **Analyzers**
   - lucene.standard (general)
   - lucene.english (stemming)
   - lucene.keyword (exact)
   - Language-specific (40+)

5. **Performance**
   - Index size ≈ 10-30% of data
   - Explicit mappings recommended
   - Limit projected fields
   - Monitor query latency

### Configuration Minimale Production

```javascript
// Production search index
{
  "name": "products_search",
  "mappings": {
    "dynamic": false,  // Explicit only
    "fields": {
      "name": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "category": {
        "type": "string",
        "analyzer": "lucene.keyword"
      },
      "price": {
        "type": "number"
      }
    }
  }
}

// Production search query
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: searchText,
              path: ["name", "description"],
              fuzzy: { maxEdits: 1 }
            }
          }
        ],
        filter: [
          {
            range: {
              path: "price",
              gte: minPrice,
              lte: maxPrice
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      price: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $limit: 20 }
])
```

### Ressources

- [Atlas Search Documentation](https://www.mongodb.com/docs/atlas/atlas-search/)
- [Lucene Query Syntax](https://lucene.apache.org/core/documentation.html)
- [Atlas Search Tutorials](https://www.mongodb.com/docs/atlas/atlas-search/tutorials/)

---


⏭️ [Atlas Data Lake](/14-mongodb-atlas/10-atlas-data-lake.md)
