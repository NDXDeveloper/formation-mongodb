🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.8 Atlas Search et Lucene

## Introduction

**MongoDB Atlas Search** est un moteur de recherche full-text intégré à MongoDB Atlas, basé sur **Apache Lucene**. Contrairement aux index text standards de MongoDB, Atlas Search offre des capacités de recherche avancées : recherche floue (fuzzy), autocomplétion intelligente, recherche vectorielle, facettes dynamiques, synonymes, et bien plus.

Atlas Search est idéal pour les applications nécessitant des fonctionnalités de recherche sophistiquées comparables à Elasticsearch, mais avec l'avantage d'être nativement intégré à MongoDB sans infrastructure séparée à gérer.

---

## Architecture et concepts

### Atlas Search vs Text Index

```
┌───────────────────────────────────────────────────────────┐
│                 Text Index (MongoDB natif)                │
│                                                           │
│  • Index intégré dans mongod                              │
│  • Stemming et stop words basiques                        │
│  • Score de pertinence simple                             │
│  • 1 index text par collection                            │
│  • Pas de typo tolerance                                  │
│  • Pas d'autocomplete optimisé                            │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│            Atlas Search (Lucene-based)                    │
│                                                           │
│  • Service séparé avec synchronisation automatique        │
│  • Analyseurs Lucene complets                             │
│  • Scoring BM25 (Best Match 25)                           │
│  • Index multiples par collection                         │
│  • Fuzzy search (typo tolerance)                          │
│  • Autocomplete avec ngrams                               │
│  • Facets, highlighting, synonyms                         │
│  • Recherche vectorielle (embeddings)                     │
│  • Compound queries complexes                             │
└───────────────────────────────────────────────────────────┘
```

### Workflow de synchronisation

```javascript
// Les données sont automatiquement synchronisées
MongoDB Collection → Atlas Search Index → Lucene Documents

// Latence de synchronisation : ~1-2 secondes
```

---

## Configuration des index Atlas Search

### Index basique

```javascript
// Configuration via Atlas UI ou API
{
  "mappings": {
    "dynamic": true  // Index tous les champs automatiquement
  }
}

// Ou index explicite
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": {
        "type": "string",
        "analyzer": "lucene.standard"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "price": {
        "type": "number"
      },
      "category": {
        "type": "string",
        "analyzer": "lucene.keyword"
      }
    }
  }
}
```

### Analyseurs disponibles

```javascript
// Analyseurs Lucene standard
const analyzers = {
  // Standard
  "lucene.standard": "Tokenisation standard, lowercase",
  "lucene.simple": "Tokenise sur non-lettres, lowercase",
  "lucene.whitespace": "Tokenise sur espaces seulement",
  "lucene.keyword": "Pas de tokenisation (texte exact)",

  // Langues
  "lucene.english": "Stemming anglais, stop words",
  "lucene.french": "Stemming français, stop words",
  "lucene.german": "Stemming allemand",
  // ... 30+ langues

  // Spécialisés
  "lucene.fingerprint": "Normalisation et dédupe",
  "lucene.pattern": "Tokenisation par pattern regex"
};

// Configuration avec analyseur personnalisé
{
  "mappings": {
    "fields": {
      "email": {
        "type": "string",
        "analyzer": "lucene.keyword",  // Exact match
        "searchAnalyzer": "lucene.standard"
      },
      "content": {
        "type": "string",
        "analyzer": "lucene.english",
        "searchAnalyzer": "lucene.english"
      }
    }
  }
}
```

### Types de champs avancés

```javascript
{
  "mappings": {
    "fields": {
      // Autocomplete avec edge ngrams
      "productName": {
        "type": "autocomplete",
        "analyzer": "lucene.standard",
        "tokenization": "edgeGram",
        "minGrams": 2,
        "maxGrams": 15,
        "foldDiacritics": true
      },

      // Vecteurs pour recherche sémantique
      "embedding": {
        "type": "knnVector",
        "dimensions": 768,
        "similarity": "cosine"
      },

      // Géospatial
      "location": {
        "type": "geo"
      },

      // Date
      "publishedAt": {
        "type": "date"
      },

      // Nested objects
      "reviews": {
        "type": "document",
        "dynamic": true,
        "fields": {
          "rating": { "type": "number" },
          "comment": {
            "type": "string",
            "analyzer": "lucene.english"
          }
        }
      }
    }
  }
}
```

---

## Opérateurs de recherche Atlas Search

### $search - Opérateur principal

```javascript
// Recherche texte simple
db.products.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "laptop gaming",
        path: ["name", "description"]
      }
    }
  }
]);

// Recherche floue (typo tolerance)
db.products.aggregate([
  {
    $search: {
      index: "products_search",
      text: {
        query: "laptp",  // Typo intentionnelle
        path: "name",
        fuzzy: {
          maxEdits: 2,
          prefixLength: 2,
          maxExpansions: 100
        }
      }
    }
  }
]);

// Autocomplete
db.products.aggregate([
  {
    $search: {
      index: "products_autocomplete",
      autocomplete: {
        query: "lapt",
        path: "name",
        tokenOrder: "sequential",
        fuzzy: {
          maxEdits: 1
        }
      }
    }
  }
]);

// Recherche avec scoring
db.products.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "laptop",
        path: "name",
        score: {
          boost: { value: 3 }
        }
      }
    }
  },
  {
    $addFields: {
      searchScore: { $meta: "searchScore" }
    }
  }
]);
```

### Compound queries

```javascript
// Combiner plusieurs critères
db.products.aggregate([
  {
    $search: {
      index: "products_search",
      compound: {
        must: [
          {
            text: {
              query: "laptop",
              path: "name"
            }
          }
        ],
        should: [
          {
            text: {
              query: "gaming",
              path: "tags",
              score: { boost: { value: 2 } }
            }
          }
        ],
        mustNot: [
          {
            text: {
              query: "refurbished",
              path: "condition"
            }
          }
        ],
        filter: [
          {
            range: {
              path: "price",
              gte: 500,
              lte: 2000
            }
          },
          {
            equals: {
              path: "inStock",
              value: true
            }
          }
        ]
      }
    }
  }
]);
```

### Facets (agrégations)

```javascript
// Facettes dynamiques
db.products.aggregate([
  {
    $searchMeta: {
      index: "products_search",
      facet: {
        operator: {
          text: {
            query: "laptop",
            path: ["name", "description"]
          }
        },
        facets: {
          // Facette par catégorie
          categories: {
            type: "string",
            path: "category",
            numBuckets: 10
          },

          // Facette par prix
          priceRanges: {
            type: "number",
            path: "price",
            boundaries: [0, 500, 1000, 2000, 5000],
            default: "other"
          },

          // Facette par date
          publishedYear: {
            type: "date",
            path: "publishedAt",
            boundaries: [
              new Date("2020-01-01"),
              new Date("2021-01-01"),
              new Date("2022-01-01"),
              new Date("2023-01-01")
            ]
          }
        }
      }
    }
  }
]);
```

---

## Cas d'usage avancés

### Cas 1 : E-commerce avec recherche intelligente

```javascript
class AtlasSearchEcommerce {
  constructor(db) {
    this.db = db;
    this.products = db.collection('products');
  }

  async setupIndexes() {
    // Configuration via Atlas API ou UI
    const searchIndexConfig = {
      name: "products_search",
      mappings: {
        dynamic: false,
        fields: {
          name: {
            type: "string",
            analyzer: "lucene.standard"
          },
          description: {
            type: "string",
            analyzer: "lucene.english"
          },
          brand: {
            type: "string",
            analyzer: "lucene.keyword"
          },
          category: {
            type: "string",
            analyzer: "lucene.keyword"
          },
          tags: {
            type: "string",
            analyzer: "lucene.standard"
          },
          price: {
            type: "number"
          },
          rating: {
            type: "number"
          },
          inStock: {
            type: "boolean"
          },
          reviews: {
            type: "document",
            fields: {
              text: {
                type: "string",
                analyzer: "lucene.english"
              },
              rating: {
                type: "number"
              }
            }
          }
        }
      }
    };

    // Autocomplete index
    const autocompleteIndexConfig = {
      name: "products_autocomplete",
      mappings: {
        fields: {
          name: {
            type: "autocomplete",
            analyzer: "lucene.standard",
            tokenization: "edgeGram",
            minGrams: 2,
            maxGrams: 15,
            foldDiacritics: true
          }
        }
      }
    };

    console.log('Search indexes configured. Apply via Atlas UI or API.');
  }

  async searchProducts(query, options = {}) {
    const {
      category = null,
      priceMin = null,
      priceMax = null,
      brands = [],
      minRating = 0,
      inStock = null,
      sortBy = 'relevance',
      page = 1,
      limit = 20
    } = options;

    const pipeline = [];

    // Construire compound query
    const compoundQuery = {
      must: [
        {
          text: {
            query: query,
            path: ["name", "description", "tags"],
            fuzzy: {
              maxEdits: 2,
              prefixLength: 3
            },
            score: {
              boost: {
                path: "name",
                value: 3
              }
            }
          }
        }
      ],
      should: [],
      filter: []
    };

    // Filtres
    if (category) {
      compoundQuery.filter.push({
        equals: {
          path: "category",
          value: category
        }
      });
    }

    if (priceMin !== null || priceMax !== null) {
      const rangeFilter = { path: "price" };
      if (priceMin !== null) rangeFilter.gte = priceMin;
      if (priceMax !== null) rangeFilter.lte = priceMax;

      compoundQuery.filter.push({ range: rangeFilter });
    }

    if (brands.length > 0) {
      compoundQuery.filter.push({
        in: {
          path: "brand",
          value: brands
        }
      });
    }

    if (minRating > 0) {
      compoundQuery.filter.push({
        range: {
          path: "rating",
          gte: minRating
        }
      });
    }

    if (inStock === true) {
      compoundQuery.filter.push({
        equals: {
          path: "inStock",
          value: true
        }
      });
    }

    // Boost produits populaires
    compoundQuery.should.push({
      range: {
        path: "salesCount",
        gte: 100,
        score: { boost: { value: 1.5 } }
      }
    });

    // Boost produits avec bonnes reviews
    compoundQuery.should.push({
      range: {
        path: "rating",
        gte: 4.5,
        score: { boost: { value: 1.2 } }
      }
    });

    // Stage de recherche
    pipeline.push({
      $search: {
        index: "products_search",
        compound: compoundQuery,
        highlight: {
          path: ["name", "description"]
        }
      }
    });

    // Score
    pipeline.push({
      $addFields: {
        searchScore: { $meta: "searchScore" },
        highlights: { $meta: "searchHighlights" }
      }
    });

    // Tri personnalisé
    if (sortBy === 'price_asc') {
      pipeline.push({ $sort: { price: 1 } });
    } else if (sortBy === 'price_desc') {
      pipeline.push({ $sort: { price: -1 } });
    } else if (sortBy === 'rating') {
      pipeline.push({ $sort: { rating: -1, searchScore: -1 } });
    } else {
      // Par défaut: pertinence
      pipeline.push({ $sort: { searchScore: -1 } });
    }

    // Pagination
    pipeline.push(
      { $skip: (page - 1) * limit },
      { $limit: limit }
    );

    // Projection
    pipeline.push({
      $project: {
        name: 1,
        brand: 1,
        price: 1,
        rating: 1,
        reviewCount: 1,
        image: 1,
        inStock: 1,
        category: 1,
        searchScore: 1,
        highlights: 1
      }
    });

    const results = await this.products.aggregate(pipeline).toArray();

    // Count total avec $searchMeta
    const countPipeline = [
      {
        $searchMeta: {
          index: "products_search",
          compound: compoundQuery,
          count: { type: "total" }
        }
      }
    ];

    const [countResult] = await this.products.aggregate(countPipeline).toArray();
    const total = countResult?.count?.total || 0;

    return {
      results,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit)
      }
    };
  }

  async autocomplete(prefix, limit = 10) {
    return await this.products.aggregate([
      {
        $search: {
          index: "products_autocomplete",
          autocomplete: {
            query: prefix,
            path: "name",
            tokenOrder: "sequential",
            fuzzy: {
              maxEdits: 1,
              prefixLength: 2
            }
          }
        }
      },
      { $limit: limit },
      {
        $project: {
          name: 1,
          brand: 1,
          image: 1,
          price: 1,
          score: { $meta: "searchScore" }
        }
      }
    ]).toArray();
  }

  async searchWithFacets(query) {
    // Recherche + facettes en une seule requête
    const [facetsResult] = await this.products.aggregate([
      {
        $searchMeta: {
          index: "products_search",
          facet: {
            operator: {
              text: {
                query: query,
                path: ["name", "description"],
                fuzzy: { maxEdits: 2 }
              }
            },
            facets: {
              categories: {
                type: "string",
                path: "category",
                numBuckets: 20
              },
              brands: {
                type: "string",
                path: "brand",
                numBuckets: 30
              },
              priceRanges: {
                type: "number",
                path: "price",
                boundaries: [0, 100, 250, 500, 1000, 2000, 5000],
                default: "other"
              },
              ratings: {
                type: "number",
                path: "rating",
                boundaries: [0, 3, 4, 4.5, 5]
              }
            }
          }
        }
      }
    ]).toArray();

    // Recherche des résultats
    const results = await this.searchProducts(query, { limit: 20 });

    return {
      ...results,
      facets: facetsResult?.facet || {}
    };
  }

  async didYouMean(query) {
    // Recherche avec typos pour suggestions
    const results = await this.products.aggregate([
      {
        $search: {
          index: "products_search",
          text: {
            query: query,
            path: "name",
            fuzzy: {
              maxEdits: 2,
              prefixLength: 1
            }
          }
        }
      },
      { $limit: 5 },
      {
        $project: {
          name: 1,
          score: { $meta: "searchScore" }
        }
      }
    ]).toArray();

    if (results.length === 0) {
      return { hasSuggestion: false };
    }

    // Extraire mots uniques
    const suggestions = new Set();
    results.forEach(r => {
      const words = r.name.toLowerCase().split(/\s+/);
      words.forEach(w => {
        if (w.length > 3) suggestions.add(w);
      });
    });

    return {
      hasSuggestion: true,
      suggestions: Array.from(suggestions).slice(0, 5)
    };
  }

  async similarProducts(productId, limit = 10) {
    // Trouver produits similaires
    const product = await this.products.findOne({ _id: productId });
    if (!product) throw new Error('Product not found');

    // Construire query basée sur attributs du produit
    const searchTerms = [
      product.name,
      product.brand,
      ...product.tags
    ].join(' ');

    return await this.products.aggregate([
      {
        $search: {
          index: "products_search",
          compound: {
            must: [
              {
                text: {
                  query: searchTerms,
                  path: ["name", "tags"]
                }
              }
            ],
            should: [
              {
                equals: {
                  path: "category",
                  value: product.category,
                  score: { boost: { value: 2 } }
                }
              },
              {
                equals: {
                  path: "brand",
                  value: product.brand,
                  score: { boost: { value: 1.5 } }
                }
              }
            ],
            mustNot: [
              {
                equals: {
                  path: "_id",
                  value: productId
                }
              }
            ]
          }
        }
      },
      { $limit: limit },
      {
        $project: {
          name: 1,
          brand: 1,
          price: 1,
          image: 1,
          rating: 1,
          score: { $meta: "searchScore" }
        }
      }
    ]).toArray();
  }

  async searchInReviews(query, productId = null) {
    // Rechercher dans les avis clients
    const matchStage = {
      compound: {
        must: [
          {
            text: {
              query: query,
              path: "reviews.text",
              fuzzy: { maxEdits: 1 }
            }
          }
        ]
      }
    };

    if (productId) {
      matchStage.compound.filter = [
        {
          equals: {
            path: "_id",
            value: productId
          }
        }
      ];
    }

    return await this.products.aggregate([
      {
        $search: {
          index: "products_search",
          ...matchStage,
          highlight: {
            path: "reviews.text",
            maxCharsToExamine: 500
          }
        }
      },
      { $unwind: "$reviews" },
      {
        $addFields: {
          highlights: { $meta: "searchHighlights" }
        }
      },
      {
        $match: {
          "reviews.text": { $regex: new RegExp(query, 'i') }
        }
      },
      { $limit: 20 },
      {
        $project: {
          productName: "$name",
          review: "$reviews",
          highlights: 1
        }
      }
    ]).toArray();
  }
}

// Utilisation
const search = new AtlasSearchEcommerce(db);
await search.setupIndexes();

// Recherche avec typo tolerance
const results = await search.searchProducts('laptp gamin', {
  category: 'Electronics',
  priceMax: 2000,
  minRating: 4,
  inStock: true
});

console.log('Results:', results.results);
console.log('Highlights:', results.results[0]?.highlights);

// Autocomplétion
const suggestions = await search.autocomplete('iph');
console.log('Autocomplete:', suggestions);

// Facettes
const withFacets = await search.searchWithFacets('laptop');
console.log('Categories:', withFacets.facets.categories);
console.log('Price ranges:', withFacets.facets.priceRanges);

// Did you mean
const didYouMean = await search.didYouMean('lpatop');
console.log('Suggestions:', didYouMean.suggestions);

// Produits similaires
const similar = await search.similarProducts(productId);
console.log('Similar products:', similar);
```

### Cas 2 : Recherche vectorielle pour recommandations

```javascript
class VectorSearchEngine {
  constructor(db) {
    this.db = db;
    this.products = db.collection('products');
    this.articles = db.collection('articles');
  }

  async setupVectorIndex() {
    // Configuration index vectoriel
    const vectorIndexConfig = {
      name: "vector_search",
      mappings: {
        fields: {
          embedding: {
            type: "knnVector",
            dimensions: 768,  // Dimension des embeddings (ex: BERT)
            similarity: "cosine"  // ou "euclidean", "dotProduct"
          },
          category: {
            type: "string",
            analyzer: "lucene.keyword"
          },
          tags: {
            type: "string"
          }
        }
      }
    };

    console.log('Vector index config ready. Apply via Atlas UI.');
  }

  async generateEmbedding(text) {
    // En production, utiliser API d'embeddings (OpenAI, Cohere, etc.)
    // Ici, simulation
    const mockEmbedding = Array.from(
      { length: 768 },
      () => Math.random()
    );
    return mockEmbedding;
  }

  async indexProductWithEmbedding(product) {
    // Générer embedding du texte
    const textToEmbed = [
      product.name,
      product.description,
      product.brand,
      ...product.tags
    ].join(' ');

    const embedding = await this.generateEmbedding(textToEmbed);

    // Stocker avec embedding
    await this.products.updateOne(
      { _id: product._id },
      { $set: { embedding } }
    );

    return embedding;
  }

  async semanticSearch(query, options = {}) {
    const {
      category = null,
      limit = 10,
      minScore = 0.7
    } = options;

    // Générer embedding de la requête
    const queryEmbedding = await this.generateEmbedding(query);

    // Recherche vectorielle
    const pipeline = [
      {
        $search: {
          index: "vector_search",
          knnBeta: {
            vector: queryEmbedding,
            path: "embedding",
            k: limit * 2,  // Chercher plus pour filtrer ensuite
            filter: category ? {
              equals: {
                path: "category",
                value: category
              }
            } : undefined
          }
        }
      },
      {
        $addFields: {
          similarityScore: { $meta: "searchScore" }
        }
      },
      {
        $match: {
          similarityScore: { $gte: minScore }
        }
      },
      { $limit: limit },
      {
        $project: {
          name: 1,
          description: 1,
          image: 1,
          price: 1,
          category: 1,
          similarityScore: 1
        }
      }
    ];

    return await this.products.aggregate(pipeline).toArray();
  }

  async hybridSearch(query, options = {}) {
    const {
      textWeight = 0.5,
      vectorWeight = 0.5,
      limit = 20
    } = options;

    // Générer embedding
    const queryEmbedding = await this.generateEmbedding(query);

    // Recherche hybride (texte + vecteur)
    return await this.products.aggregate([
      {
        $search: {
          index: "hybrid_search",
          compound: {
            should: [
              // Recherche texte
              {
                text: {
                  query: query,
                  path: ["name", "description"],
                  score: {
                    boost: { value: textWeight }
                  }
                }
              },
              // Recherche vectorielle
              {
                knnBeta: {
                  vector: queryEmbedding,
                  path: "embedding",
                  k: 100,
                  score: {
                    boost: { value: vectorWeight }
                  }
                }
              }
            ]
          }
        }
      },
      {
        $addFields: {
          combinedScore: { $meta: "searchScore" }
        }
      },
      { $sort: { combinedScore: -1 } },
      { $limit: limit }
    ]).toArray();
  }

  async findSimilarProducts(productId, limit = 10) {
    // Trouver produits similaires via embedding
    const product = await this.products.findOne({ _id: productId });
    if (!product || !product.embedding) {
      throw new Error('Product or embedding not found');
    }

    return await this.products.aggregate([
      {
        $search: {
          index: "vector_search",
          knnBeta: {
            vector: product.embedding,
            path: "embedding",
            k: limit + 1  // +1 pour exclure le produit lui-même
          }
        }
      },
      {
        $match: {
          _id: { $ne: productId }
        }
      },
      { $limit: limit },
      {
        $addFields: {
          similarityScore: { $meta: "searchScore" }
        }
      },
      {
        $project: {
          name: 1,
          price: 1,
          image: 1,
          similarityScore: 1
        }
      }
    ]).toArray();
  }

  async recommendBasedOnHistory(userHistory, limit = 10) {
    // Recommandations basées sur historique utilisateur
    // userHistory = array of product IDs

    // Récupérer embeddings des produits vus
    const viewedProducts = await this.products.find({
      _id: { $in: userHistory }
    }).toArray();

    if (viewedProducts.length === 0) {
      return [];
    }

    // Calculer embedding moyen
    const avgEmbedding = this.averageEmbeddings(
      viewedProducts.map(p => p.embedding)
    );

    // Chercher produits similaires au profil
    return await this.products.aggregate([
      {
        $search: {
          index: "vector_search",
          knnBeta: {
            vector: avgEmbedding,
            path: "embedding",
            k: limit * 2
          }
        }
      },
      {
        $match: {
          _id: { $nin: userHistory }  // Exclure déjà vus
        }
      },
      { $limit: limit },
      {
        $addFields: {
          recommendationScore: { $meta: "searchScore" }
        }
      }
    ]).toArray();
  }

  averageEmbeddings(embeddings) {
    const dims = embeddings[0].length;
    const avg = new Array(dims).fill(0);

    embeddings.forEach(emb => {
      emb.forEach((val, i) => {
        avg[i] += val;
      });
    });

    return avg.map(val => val / embeddings.length);
  }

  async clusterProducts(limit = 1000) {
    // Récupérer embeddings
    const products = await this.products.find({
      embedding: { $exists: true }
    })
      .limit(limit)
      .toArray();

    // Clustering simple avec k-means (simplification)
    // En production, utiliser bibliothèque ML appropriée
    const clusters = this.simpleKMeans(
      products.map(p => p.embedding),
      10  // 10 clusters
    );

    return {
      clusterCount: clusters.length,
      products: products.map((p, i) => ({
        ...p,
        cluster: clusters[i]
      }))
    };
  }

  simpleKMeans(vectors, k) {
    // Implémentation simplifiée
    // En production, utiliser ML.js, tensorflow.js, etc.
    return vectors.map(() => Math.floor(Math.random() * k));
  }
}

// Utilisation
const vectorSearch = new VectorSearchEngine(db);
await vectorSearch.setupVectorIndex();

// Indexer produits avec embeddings
for (const product of products) {
  await vectorSearch.indexProductWithEmbedding(product);
}

// Recherche sémantique
const semantic = await vectorSearch.semanticSearch(
  'high performance computing device',
  { category: 'Electronics', limit: 10 }
);
console.log('Semantic results:', semantic);

// Recherche hybride
const hybrid = await vectorSearch.hybridSearch(
  'gaming laptop',
  { textWeight: 0.4, vectorWeight: 0.6 }
);

// Recommandations basées sur historique
const recommendations = await vectorSearch.recommendBasedOnHistory(
  [productId1, productId2, productId3],
  10
);
console.log('Recommendations:', recommendations);
```

### Cas 3 : Recherche multi-tenant avec isolation

```javascript
class MultiTenantSearch {
  constructor(db) {
    this.db = db;
    this.documents = db.collection('documents');
  }

  async setupMultiTenantIndex() {
    // Index avec champs tenant
    const indexConfig = {
      name: "multi_tenant_search",
      mappings: {
        fields: {
          tenantId: {
            type: "string",
            analyzer: "lucene.keyword"
          },
          organizationId: {
            type: "string",
            analyzer: "lucene.keyword"
          },
          title: {
            type: "string",
            analyzer: "lucene.standard"
          },
          content: {
            type: "string",
            analyzer: "lucene.english"
          },
          tags: {
            type: "string"
          },
          permissions: {
            type: "document",
            fields: {
              users: {
                type: "string",
                analyzer: "lucene.keyword"
              },
              groups: {
                type: "string",
                analyzer: "lucene.keyword"
              },
              level: {
                type: "string",
                analyzer: "lucene.keyword"
              }
            }
          },
          createdAt: {
            type: "date"
          }
        }
      }
    };

    console.log('Multi-tenant index config ready.');
  }

  async searchWithPermissions(query, user, options = {}) {
    const {
      tenantId,
      organizationId = null,
      tags = [],
      dateFrom = null,
      dateTo = null,
      page = 1,
      limit = 20
    } = options;

    // Construire query avec isolation tenant
    const compoundQuery = {
      must: [
        // Recherche texte
        {
          text: {
            query: query,
            path: ["title", "content"],
            fuzzy: { maxEdits: 2 }
          }
        },
        // Isolation tenant (sécurité critique)
        {
          equals: {
            path: "tenantId",
            value: tenantId
          }
        }
      ],
      should: [],
      filter: []
    };

    // Filtre organisation si spécifié
    if (organizationId) {
      compoundQuery.filter.push({
        equals: {
          path: "organizationId",
          value: organizationId
        }
      });
    }

    // Permissions utilisateur
    compoundQuery.filter.push({
      compound: {
        should: [
          // Documents publics
          {
            equals: {
              path: "permissions.level",
              value: "public"
            }
          },
          // Documents où user est dans permissions
          {
            equals: {
              path: "permissions.users",
              value: user.id
            }
          },
          // Documents où groupe user est dans permissions
          {
            in: {
              path: "permissions.groups",
              value: user.groups
            }
          },
          // Documents créés par l'utilisateur
          {
            equals: {
              path: "createdBy",
              value: user.id
            }
          }
        ],
        minimumShouldMatch: 1
      }
    });

    // Filtres additionnels
    if (tags.length > 0) {
      compoundQuery.filter.push({
        in: {
          path: "tags",
          value: tags
        }
      });
    }

    if (dateFrom || dateTo) {
      const dateFilter = { path: "createdAt" };
      if (dateFrom) dateFilter.gte = dateFrom;
      if (dateTo) dateFilter.lte = dateTo;
      compoundQuery.filter.push({ range: dateFilter });
    }

    // Exécuter recherche
    const pipeline = [
      {
        $search: {
          index: "multi_tenant_search",
          compound: compoundQuery,
          highlight: {
            path: ["title", "content"]
          }
        }
      },
      {
        $addFields: {
          searchScore: { $meta: "searchScore" },
          highlights: { $meta: "searchHighlights" }
        }
      },
      { $sort: { searchScore: -1 } },
      { $skip: (page - 1) * limit },
      { $limit: limit },
      {
        $project: {
          title: 1,
          excerpt: 1,
          tags: 1,
          createdBy: 1,
          createdAt: 1,
          organizationId: 1,
          searchScore: 1,
          highlights: 1,
          // Ne pas exposer permissions complètes
          canEdit: {
            $cond: [
              { $eq: ["$createdBy", user.id] },
              true,
              false
            ]
          }
        }
      }
    ];

    const results = await this.documents.aggregate(pipeline).toArray();

    // Count avec même filtres de sécurité
    const [countResult] = await this.documents.aggregate([
      {
        $searchMeta: {
          index: "multi_tenant_search",
          compound: compoundQuery,
          count: { type: "total" }
        }
      }
    ]).toArray();

    const total = countResult?.count?.total || 0;

    return {
      results,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit)
      }
    };
  }

  async searchAcrossOrganizations(query, user, allowedOrgs) {
    // Recherche à travers plusieurs organisations (admin use case)
    return await this.documents.aggregate([
      {
        $search: {
          index: "multi_tenant_search",
          compound: {
            must: [
              {
                text: {
                  query: query,
                  path: ["title", "content"]
                }
              },
              {
                equals: {
                  path: "tenantId",
                  value: user.tenantId
                }
              }
            ],
            filter: [
              {
                in: {
                  path: "organizationId",
                  value: allowedOrgs
                }
              }
            ]
          }
        }
      },
      {
        $addFields: {
          score: { $meta: "searchScore" }
        }
      },
      {
        $facet: {
          results: [
            { $sort: { score: -1 } },
            { $limit: 50 }
          ],
          byOrganization: [
            {
              $group: {
                _id: "$organizationId",
                count: { $sum: 1 },
                avgScore: { $avg: "$score" }
              }
            },
            { $sort: { count: -1 } }
          ]
        }
      }
    ]).toArray();
  }

  async auditSearch(query, user, tenantId, results) {
    // Logger recherche pour audit/compliance
    await this.db.collection('search_audit').insertOne({
      query,
      userId: user.id,
      tenantId,
      resultsCount: results.length,
      timestamp: new Date(),
      userGroups: user.groups,
      resultIds: results.slice(0, 10).map(r => r._id)
    });
  }

  async getSearchAnalytics(tenantId, startDate, endDate) {
    // Analytics par tenant
    return await this.db.collection('search_audit').aggregate([
      {
        $match: {
          tenantId,
          timestamp: {
            $gte: startDate,
            $lte: endDate
          }
        }
      },
      {
        $facet: {
          topQueries: [
            {
              $group: {
                _id: { $toLower: "$query" },
                count: { $sum: 1 },
                avgResults: { $avg: "$resultsCount" }
              }
            },
            { $sort: { count: -1 } },
            { $limit: 20 }
          ],
          activeUsers: [
            {
              $group: {
                _id: "$userId",
                searchCount: { $sum: 1 }
              }
            },
            { $sort: { searchCount: -1 } },
            { $limit: 10 }
          ],
          searchesByDay: [
            {
              $group: {
                _id: {
                  $dateToString: {
                    format: "%Y-%m-%d",
                    date: "$timestamp"
                  }
                },
                count: { $sum: 1 }
              }
            },
            { $sort: { _id: 1 } }
          ],
          zeroResultQueries: [
            {
              $match: { resultsCount: 0 }
            },
            {
              $group: {
                _id: "$query",
                count: { $sum: 1 }
              }
            },
            { $sort: { count: -1 } },
            { $limit: 10 }
          ]
        }
      }
    ]).toArray();
  }
}

// Utilisation
const multiTenant = new MultiTenantSearch(db);
await multiTenant.setupMultiTenantIndex();

// Recherche avec permissions utilisateur
const user = {
  id: 'user-123',
  tenantId: 'tenant-abc',
  groups: ['engineering', 'managers']
};

const results = await multiTenant.searchWithPermissions(
  'quarterly report',
  user,
  {
    tenantId: user.tenantId,
    organizationId: 'org-456',
    tags: ['finance'],
    dateFrom: new Date('2024-01-01')
  }
);

// Audit
await multiTenant.auditSearch(
  'quarterly report',
  user,
  user.tenantId,
  results.results
);

// Analytics
const analytics = await multiTenant.getSearchAnalytics(
  user.tenantId,
  new Date('2024-11-01'),
  new Date('2024-12-01')
);
console.log('Top queries:', analytics[0].topQueries);
console.log('Zero results:', analytics[0].zeroResultQueries);
```

---

## Performance et optimisations

### Sizing et performance

```javascript
// Atlas Search utilise index Lucene séparés
// Sizing recommandations:
const sizing = {
  indexSize: "~30-40% de la taille des données",
  ramRecommended: "1GB RAM par 5GB d'index",
  searchNodes: "Dédiés pour haute charge (M30+)",
  syncLatency: "1-2 secondes (normal)",

  bestPractices: {
    // Limiter champs indexés
    dynamicMapping: false,
    indexOnlyNeeded: true,

    // Projections pour réduire résultats
    useProjection: true,
    returnOnlyNeeded: ["_id", "title", "image"],

    // Pagination
    maxPageSize: 100,
    useSkipSparingly: "Skip coûteux, préférer search_after"
  }
};
```

### Monitoring

```javascript
class AtlasSearchMonitoring {
  async getIndexStats(indexName) {
    // Via Atlas API
    const stats = await fetch(
      `https://cloud.mongodb.com/api/atlas/v1.0/groups/${groupId}/clusters/${clusterName}/fts/indexes/${indexName}/stats`,
      {
        headers: {
          'Authorization': `Bearer ${apiToken}`
        }
      }
    ).then(r => r.json());

    return {
      indexSize: stats.indexSize,
      documentCount: stats.documentCount,
      avgDocumentSize: stats.avgDocumentSize,
      queryStats: {
        queriesPerSecond: stats.queriesPerSecond,
        avgQueryLatency: stats.avgQueryLatency
      }
    };
  }

  async analyzeSlowQueries(threshold = 1000) {
    // Logger queries lentes
    const slowQueries = [];

    const searchWithTiming = async (pipeline) => {
      const start = Date.now();
      const results = await db.collection.aggregate(pipeline).toArray();
      const duration = Date.now() - start;

      if (duration > threshold) {
        slowQueries.push({
          pipeline,
          duration,
          resultCount: results.length,
          timestamp: new Date()
        });
      }

      return results;
    };

    return { slowQueries };
  }
}
```

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Désactiver dynamic mapping en production
{
  "mappings": {
    "dynamic": false,  // ✅ Contrôle explicite
    "fields": { /* ... */ }
  }
}

// 2. Utiliser projections
db.collection.aggregate([
  { $search: { /* ... */ } },
  {
    $project: {
      _id: 1,
      title: 1,
      score: { $meta: "searchScore" }
      // Seulement champs nécessaires
    }
  }
]);

// 3. Limiter résultats
db.collection.aggregate([
  { $search: { /* ... */ } },
  { $limit: 100 }  // ✅ Toujours limiter
]);

// 4. Utiliser compound queries pour filtres
{
  $search: {
    compound: {
      must: [/* recherche */],
      filter: [/* filtres non-scorés */]
    }
  }
}

// 5. Monitorer usage et coûts
// Atlas Search est facturé séparément
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas utiliser dynamic: true en production
// ❌ Index tout, coûteux
{
  "mappings": { "dynamic": true }
}

// 2. Ne pas retourner tous les champs
// ❌ Lent et coûteux
db.collection.aggregate([
  { $search: { /* ... */ } }
  // Pas de $project
]);

// 3. Ne pas faire skip important
// ❌ Très coûteux
db.collection.aggregate([
  { $search: { /* ... */ } },
  { $skip: 10000 },  // Lent!
  { $limit: 20 }
]);

// 4. Ne pas indexer champs énormes
// ❌ Performance impact
{
  "massiveTextField": {
    "type": "string"  // Plusieurs MB
  }
}

// 5. Ne pas négliger les coûts
// Atlas Search = coût supplémentaire
// Monitorer et optimiser
```

---

## Conclusion

Atlas Search offre des capacités de recherche enterprise-grade :
- ✅ **Fuzzy search** (typo tolerance)
- ✅ **Autocomplete optimisé** avec ngrams
- ✅ **Recherche vectorielle** (embeddings, semantic search)
- ✅ **Facets dynamiques** pour filtres
- ✅ **Highlighting** automatique
- ✅ **Compound queries** sophistiquées
- ✅ **Performance Lucene** éprouvée

**Points clés à retenir :**
1. Basé sur Apache Lucene (battle-tested)
2. Service géré séparément (sync automatique)
3. Coût supplémentaire vs text index standard
4. Mapping explicite recommandé en production
5. Excellent pour e-commerce, recherche sémantique, multi-tenant

**Cas d'usage idéaux :**
- E-commerce avec autocomplete et fuzzy
- Systèmes de recommandation (vectoriel)
- Recherche documentaire sophistiquée
- Multi-tenant avec permissions
- Applications nécessitant typo tolerance

**Alternatives à considérer :**
- Text Index MongoDB si besoins simples
- Elasticsearch si infra déjà en place
- Atlas Search si sur Atlas (recommended)

---


⏭️ [Vector Search et IA](/16-fonctionnalites-avancees/09-vector-search-ia.md)
