🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.7 Recherche Full-Text avancée

## Introduction

La **recherche Full-Text** (recherche en texte intégral) permet d'effectuer des recherches linguistiquement intelligentes dans des champs textuels. MongoDB offre des capacités de recherche texte intégrées avec support multilingue, stemming, stop words, et scoring de pertinence.

Cette fonctionnalité est essentielle pour les moteurs de recherche, e-commerce, systèmes de gestion de contenu, bases de connaissances, ou toute application nécessitant des recherches textuelles sophistiquées.

---

## Concepts fondamentaux

### Index text

```javascript
// Index text simple sur un champ
await db.collection('articles').createIndex({ content: "text" });

// Index text sur plusieurs champs
await db.collection('products').createIndex({
  name: "text",
  description: "text",
  tags: "text"
});

// Index text avec pondération (weights)
await db.collection('articles').createIndex(
  {
    title: "text",
    content: "text",
    tags: "text"
  },
  {
    weights: {
      title: 10,      // Titre 10x plus important
      content: 5,     // Contenu 5x
      tags: 3         // Tags 3x
    },
    name: "article_text_index"
  }
);

// Index text avec langue par défaut
await db.collection('articles').createIndex(
  { content: "text" },
  { default_language: "french" }
);
```

### Langues supportées

MongoDB supporte 15+ langues avec stemming et stop words :

```javascript
const supportedLanguages = [
  'danish', 'dutch', 'english', 'finnish', 'french',
  'german', 'hungarian', 'italian', 'norwegian', 'portuguese',
  'romanian', 'russian', 'spanish', 'swedish', 'turkish'
];

// Document avec langue spécifique
await db.collection('articles').insertOne({
  title: "Article en français",
  content: "Contenu de l'article...",
  language: "french"  // Champ pour override langue
});

// Index avec langue personnalisée par document
await db.collection('articles').createIndex(
  { content: "text" },
  { default_language: "english", language_override: "language" }
);
```

---

## Opérateurs de recherche

### $text - Recherche de base

```javascript
// Recherche simple
const results = await db.collection('articles').find({
  $text: { $search: "mongodb database" }
}).toArray();

// Recherche avec phrase exacte
const exactPhrase = await db.collection('articles').find({
  $text: { $search: "\"replica set\"" }  // Guillemets pour phrase exacte
}).toArray();

// Recherche avec exclusion
const excluded = await db.collection('articles').find({
  $text: { $search: "mongodb -deprecated" }  // Exclut "deprecated"
}).toArray();

// Recherche avec langue spécifique
const french = await db.collection('articles').find({
  $text: {
    $search: "base de données",
    $language: "french"
  }
}).toArray();

// Recherche case-sensitive
const caseSensitive = await db.collection('articles').find({
  $text: {
    $search: "MongoDB",
    $caseSensitive: true
  }
}).toArray();

// Recherche diacritique-sensitive
const diacriticSensitive = await db.collection('articles').find({
  $text: {
    $search: "café",
    $diacriticSensitive: true  // Distingue café de cafe
  }
}).toArray();
```

### Score de pertinence

```javascript
// Obtenir le score de pertinence
const resultsWithScore = await db.collection('articles').find(
  { $text: { $search: "mongodb replication" } },
  { score: { $meta: "textScore" } }
)
  .sort({ score: { $meta: "textScore" } })  // Trier par pertinence
  .toArray();

// Filtrer par score minimum
const relevantOnly = await db.collection('articles').find(
  {
    $text: { $search: "mongodb" },
    score: { $meta: "textScore" }
  },
  { score: { $meta: "textScore" } }
)
  .match({ score: { $gte: 1.5 } })  // Score minimum
  .sort({ score: { $meta: "textScore" } })
  .toArray();
```

---

## Cas d'usage avancés

### Cas 1 : Moteur de recherche e-commerce

```javascript
class EcommerceSearchEngine {
  constructor(db) {
    this.db = db;
    this.products = null;
  }

  async initialize() {
    this.products = this.db.collection('products');

    // Index text avec pondération stratégique
    try {
      await this.products.createIndex(
        {
          name: "text",
          brand: "text",
          description: "text",
          category: "text",
          tags: "text"
        },
        {
          weights: {
            name: 10,        // Nom très important
            brand: 8,        // Marque importante
            tags: 5,         // Tags moyennement importants
            category: 3,     // Catégorie moins importante
            description: 2   // Description moins pondérée
          },
          name: "product_search_index"
        }
      );
    } catch (error) {
      if (error.code !== 85) throw error;  // Index déjà existe
    }

    // Index secondaires pour filtres
    await this.products.createIndex({ category: 1, price: 1 });
    await this.products.createIndex({ brand: 1 });
    await this.products.createIndex({ rating: -1 });
    await this.products.createIndex({ inStock: 1 });
  }

  async search(query, options = {}) {
    const {
      category = null,
      priceMin = 0,
      priceMax = Infinity,
      brands = [],
      minRating = 0,
      inStock = false,
      sortBy = 'relevance',
      page = 1,
      limit = 20
    } = options;

    // Pipeline d'agrégation pour recherche avancée
    const pipeline = [];

    // 1. Recherche texte avec score
    pipeline.push({
      $match: {
        $text: { $search: query }
      }
    });

    pipeline.push({
      $addFields: {
        searchScore: { $meta: "textScore" }
      }
    });

    // 2. Filtres
    const filters = {
      price: { $gte: priceMin, $lte: priceMax }
    };

    if (category) filters.category = category;
    if (brands.length > 0) filters.brand = { $in: brands };
    if (minRating > 0) filters.rating = { $gte: minRating };
    if (inStock) filters.inStock = true;

    pipeline.push({ $match: filters });

    // 3. Enrichissement avec scoring personnalisé
    pipeline.push({
      $addFields: {
        finalScore: {
          $add: [
            '$searchScore',                           // Score texte de base
            { $multiply: ['$rating', 0.5] },         // +0.5 par point de rating
            { $cond: ['$inStock', 2, 0] },           // +2 si en stock
            { $cond: ['$featured', 3, 0] },          // +3 si produit vedette
            { $multiply: [                            // Bonus popularité
              { $divide: ['$salesCount', 1000] },
              1
            ]}
          ]
        }
      }
    });

    // 4. Tri
    const sortStage = {};
    switch (sortBy) {
      case 'relevance':
        sortStage.finalScore = -1;
        break;
      case 'price_asc':
        sortStage.price = 1;
        break;
      case 'price_desc':
        sortStage.price = -1;
        break;
      case 'rating':
        sortStage.rating = -1;
        break;
      case 'newest':
        sortStage.createdAt = -1;
        break;
      default:
        sortStage.finalScore = -1;
    }

    pipeline.push({ $sort: sortStage });

    // 5. Pagination
    pipeline.push({ $skip: (page - 1) * limit });
    pipeline.push({ $limit: limit });

    // 6. Projection (retourner seulement champs nécessaires)
    pipeline.push({
      $project: {
        name: 1,
        brand: 1,
        price: 1,
        originalPrice: 1,
        rating: 1,
        reviewCount: 1,
        image: 1,
        inStock: 1,
        category: 1,
        searchScore: 1,
        finalScore: 1
      }
    });

    // Exécuter recherche
    const results = await this.products.aggregate(pipeline).toArray();

    // Compter total (pour pagination)
    const countPipeline = [
      { $match: { $text: { $search: query } } },
      { $match: filters },
      { $count: 'total' }
    ];

    const countResult = await this.products.aggregate(countPipeline).toArray();
    const total = countResult[0]?.total || 0;

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

  async searchWithFacets(query, options = {}) {
    // Recherche avec facettes pour filtres dynamiques
    const { category = null } = options;

    const pipeline = [
      {
        $match: {
          $text: { $search: query }
        }
      },
      {
        $addFields: {
          searchScore: { $meta: "textScore" }
        }
      }
    ];

    if (category) {
      pipeline.push({ $match: { category } });
    }

    // Facets parallèles
    pipeline.push({
      $facet: {
        // Résultats principaux
        results: [
          { $sort: { searchScore: -1 } },
          { $limit: 20 },
          {
            $project: {
              name: 1,
              brand: 1,
              price: 1,
              rating: 1,
              image: 1,
              searchScore: 1
            }
          }
        ],

        // Facette: catégories
        categories: [
          {
            $group: {
              _id: '$category',
              count: { $sum: 1 },
              avgPrice: { $avg: '$price' }
            }
          },
          { $sort: { count: -1 } },
          { $limit: 10 }
        ],

        // Facette: marques
        brands: [
          {
            $group: {
              _id: '$brand',
              count: { $sum: 1 }
            }
          },
          { $sort: { count: -1 } },
          { $limit: 20 }
        ],

        // Facette: gamme de prix
        priceRanges: [
          {
            $bucket: {
              groupBy: '$price',
              boundaries: [0, 25, 50, 100, 200, 500, 1000],
              default: 'Other',
              output: {
                count: { $sum: 1 },
                avgRating: { $avg: '$rating' }
              }
            }
          }
        ],

        // Facette: ratings
        ratings: [
          {
            $bucket: {
              groupBy: '$rating',
              boundaries: [0, 2, 3, 4, 4.5, 5],
              default: 'Unrated',
              output: {
                count: { $sum: 1 }
              }
            }
          }
        ],

        // Statistiques globales
        stats: [
          {
            $group: {
              _id: null,
              total: { $sum: 1 },
              avgPrice: { $avg: '$price' },
              minPrice: { $min: '$price' },
              maxPrice: { $max: '$price' },
              avgRating: { $avg: '$rating' }
            }
          }
        ]
      }
    });

    const [result] = await this.products.aggregate(pipeline).toArray();
    return result;
  }

  async autocomplete(prefix, limit = 10) {
    // Autocomplétion basique avec regex
    // Note: Pour production, utiliser Atlas Search pour meilleure performance
    const regex = new RegExp(`^${prefix}`, 'i');

    return await this.products.aggregate([
      {
        $match: {
          $or: [
            { name: regex },
            { brand: regex }
          ]
        }
      },
      {
        $group: {
          _id: null,
          suggestions: {
            $addToSet: '$name'
          }
        }
      },
      {
        $project: {
          suggestions: { $slice: ['$suggestions', limit] }
        }
      }
    ]).toArray();
  }

  async getSuggestions(query) {
    // Suggestions "Did you mean?" basées sur résultats
    const results = await this.products.find(
      { $text: { $search: query } },
      { score: { $meta: "textScore" } }
    )
      .limit(5)
      .toArray();

    if (results.length === 0) {
      // Recherche fuzzy approximative
      return await this.fuzzySearch(query);
    }

    return results;
  }

  async fuzzySearch(query) {
    // Recherche approximative simple avec regex
    const terms = query.split(' ').filter(t => t.length > 2);
    const regexes = terms.map(t => new RegExp(t, 'i'));

    return await this.products.find({
      $or: [
        { name: { $in: regexes } },
        { description: { $in: regexes } }
      ]
    })
      .limit(10)
      .toArray();
  }

  async trackSearch(query, userId, resultsCount) {
    // Analytics de recherche
    await this.db.collection('search_analytics').insertOne({
      query,
      userId,
      resultsCount,
      timestamp: new Date()
    });
  }

  async getPopularSearches(limit = 10) {
    return await this.db.collection('search_analytics').aggregate([
      {
        $match: {
          timestamp: {
            $gte: new Date(Date.now() - 7 * 86400000)  // 7 jours
          }
        }
      },
      {
        $group: {
          _id: { $toLower: '$query' },
          count: { $sum: 1 },
          avgResults: { $avg: '$resultsCount' }
        }
      },
      { $sort: { count: -1 } },
      { $limit: limit },
      {
        $project: {
          query: '$_id',
          searchCount: '$count',
          avgResults: { $round: ['$avgResults', 0] },
          _id: 0
        }
      }
    ]).toArray();
  }
}

// Utilisation
const searchEngine = new EcommerceSearchEngine(db);
await searchEngine.initialize();

// Recherche basique
const results = await searchEngine.search('laptop gaming', {
  category: 'Electronics',
  priceMax: 2000,
  minRating: 4,
  inStock: true,
  sortBy: 'relevance',
  page: 1,
  limit: 20
});

console.log(`Found ${results.pagination.total} products`);
console.log('Top result:', results.results[0]);

// Recherche avec facettes
const withFacets = await searchEngine.searchWithFacets('smartphone');
console.log('Categories:', withFacets.categories);
console.log('Brands:', withFacets.brands);
console.log('Price ranges:', withFacets.priceRanges);

// Autocomplétion
const suggestions = await searchEngine.autocomplete('iph');
console.log('Suggestions:', suggestions);

// Analytics
const popular = await searchEngine.getPopularSearches(10);
console.log('Popular searches:', popular);
```

### Cas 2 : Système de recherche de contenu/blog

```javascript
class ContentSearchEngine {
  constructor(db) {
    this.db = db;
    this.articles = null;
  }

  async initialize() {
    this.articles = this.db.collection('articles');

    // Index text multilingue
    try {
      await this.articles.createIndex(
        {
          title: "text",
          content: "text",
          excerpt: "text",
          tags: "text",
          "author.name": "text"
        },
        {
          weights: {
            title: 15,
            tags: 10,
            excerpt: 5,
            "author.name": 3,
            content: 1
          },
          default_language: "french",
          language_override: "language"
        }
      );
    } catch (error) {
      if (error.code !== 85) throw error;
    }

    // Index pour filtres et tri
    await this.articles.createIndex({ publishedAt: -1 });
    await this.articles.createIndex({ category: 1, publishedAt: -1 });
    await this.articles.createIndex({ "author.id": 1 });
    await this.articles.createIndex({ tags: 1 });
  }

  async search(query, options = {}) {
    const {
      categories = [],
      tags = [],
      authors = [],
      dateFrom = null,
      dateTo = null,
      language = null,
      page = 1,
      limit = 10
    } = options;

    const pipeline = [];

    // Recherche text
    const textSearch = {
      $text: { $search: query }
    };

    if (language) {
      textSearch.$text.$language = language;
    }

    pipeline.push({ $match: textSearch });

    // Score
    pipeline.push({
      $addFields: {
        textScore: { $meta: "textScore" }
      }
    });

    // Filtres
    const filters = { status: 'published' };

    if (categories.length > 0) {
      filters.category = { $in: categories };
    }

    if (tags.length > 0) {
      filters.tags = { $in: tags };
    }

    if (authors.length > 0) {
      filters['author.id'] = { $in: authors };
    }

    if (dateFrom || dateTo) {
      filters.publishedAt = {};
      if (dateFrom) filters.publishedAt.$gte = dateFrom;
      if (dateTo) filters.publishedAt.$lte = dateTo;
    }

    pipeline.push({ $match: filters });

    // Enrichissement avec scoring avancé
    pipeline.push({
      $addFields: {
        relevanceScore: {
          $add: [
            { $multiply: ['$textScore', 5] },     // Score texte base
            { $divide: ['$views', 1000] },        // Bonus vues
            { $multiply: ['$avgRating', 2] },     // Bonus rating
            {
              $cond: [                             // Bonus récence
                {
                  $gt: [
                    '$publishedAt',
                    new Date(Date.now() - 30 * 86400000)
                  ]
                },
                5,
                0
              ]
            },
            { $size: { $ifNull: ['$comments', []] } }  // Bonus engagement
          ]
        }
      }
    });

    // Tri
    pipeline.push({ $sort: { relevanceScore: -1, publishedAt: -1 } });

    // Pagination
    pipeline.push({ $skip: (page - 1) * limit });
    pipeline.push({ $limit: limit });

    // Projection
    pipeline.push({
      $project: {
        title: 1,
        excerpt: 1,
        category: 1,
        tags: 1,
        author: 1,
        publishedAt: 1,
        featuredImage: 1,
        readTime: 1,
        views: 1,
        commentsCount: { $size: { $ifNull: ['$comments', []] } },
        textScore: 1,
        relevanceScore: 1
      }
    });

    // Exécuter
    const results = await this.articles.aggregate(pipeline).toArray();

    // Count total
    const countPipeline = [
      { $match: textSearch },
      { $match: filters },
      { $count: 'total' }
    ];

    const countResult = await this.articles.aggregate(countPipeline).toArray();
    const total = countResult[0]?.total || 0;

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

  async searchRelated(articleId, limit = 5) {
    // Trouver articles similaires basés sur tags et contenu
    const article = await this.articles.findOne({ _id: articleId });
    if (!article) throw new Error('Article not found');

    // Construire query de similarité
    const searchTerms = [
      ...article.tags,
      ...article.title.split(' ').filter(w => w.length > 3)
    ].join(' ');

    return await this.articles.aggregate([
      {
        $match: {
          _id: { $ne: articleId },
          $text: { $search: searchTerms }
        }
      },
      {
        $addFields: {
          textScore: { $meta: "textScore" },
          tagSimilarity: {
            $size: {
              $setIntersection: [article.tags, '$tags']
            }
          }
        }
      },
      {
        $addFields: {
          similarityScore: {
            $add: [
              { $multiply: ['$textScore', 2] },
              { $multiply: ['$tagSimilarity', 5] }
            ]
          }
        }
      },
      { $sort: { similarityScore: -1 } },
      { $limit: limit },
      {
        $project: {
          title: 1,
          excerpt: 1,
          category: 1,
          author: 1,
          publishedAt: 1,
          featuredImage: 1,
          similarityScore: 1
        }
      }
    ]).toArray();
  }

  async searchByContext(context, limit = 10) {
    // Recherche contextuelle (ex: "articles about X for beginners")
    // Analyser le contexte et ajuster weights
    const isBeginnerContext = /beginner|introduction|getting started|basics/i.test(context);
    const isAdvancedContext = /advanced|expert|deep dive|detailed/i.test(context);

    const difficultyFilter = {};
    if (isBeginnerContext) {
      difficultyFilter.difficulty = { $in: ['beginner', 'intermediate'] };
    } else if (isAdvancedContext) {
      difficultyFilter.difficulty = 'advanced';
    }

    return await this.articles.aggregate([
      {
        $match: {
          $text: { $search: context },
          ...difficultyFilter
        }
      },
      {
        $addFields: {
          score: { $meta: "textScore" }
        }
      },
      { $sort: { score: -1 } },
      { $limit: limit }
    ]).toArray();
  }

  async highlightMatches(articleId, query) {
    // Extraire et surligner les correspondances
    const article = await this.articles.findOne({ _id: articleId });
    if (!article) return null;

    const terms = query.toLowerCase().split(' ').filter(t => t.length > 2);
    const content = article.content;

    // Trouver snippets pertinents
    const snippets = [];
    const sentences = content.split(/[.!?]+/);

    for (const sentence of sentences) {
      const lowerSentence = sentence.toLowerCase();
      const matchCount = terms.filter(term =>
        lowerSentence.includes(term)
      ).length;

      if (matchCount > 0) {
        // Surligner les termes
        let highlighted = sentence;
        terms.forEach(term => {
          const regex = new RegExp(`(${term})`, 'gi');
          highlighted = highlighted.replace(regex, '<mark>$1</mark>');
        });

        snippets.push({
          text: highlighted.trim(),
          relevance: matchCount
        });
      }
    }

    return {
      article: {
        title: article.title,
        author: article.author
      },
      snippets: snippets
        .sort((a, b) => b.relevance - a.relevance)
        .slice(0, 3)
    };
  }

  async generateSearchSummary(query) {
    // Générer résumé des résultats de recherche
    const results = await this.articles.aggregate([
      {
        $match: {
          $text: { $search: query }
        }
      },
      {
        $addFields: {
          score: { $meta: "textScore" }
        }
      },
      {
        $facet: {
          topResults: [
            { $sort: { score: -1 } },
            { $limit: 3 },
            {
              $project: {
                title: 1,
                excerpt: 1,
                score: 1
              }
            }
          ],
          categories: [
            {
              $group: {
                _id: '$category',
                count: { $sum: 1 },
                avgScore: { $avg: '$score' }
              }
            },
            { $sort: { count: -1 } }
          ],
          timeDistribution: [
            {
              $group: {
                _id: {
                  year: { $year: '$publishedAt' },
                  month: { $month: '$publishedAt' }
                },
                count: { $sum: 1 }
              }
            },
            { $sort: { '_id.year': -1, '_id.month': -1 } },
            { $limit: 12 }
          ],
          stats: [
            {
              $group: {
                _id: null,
                total: { $sum: 1 },
                avgScore: { $avg: '$score' },
                totalViews: { $sum: '$views' }
              }
            }
          ]
        }
      }
    ]).toArray();

    return results[0];
  }
}

// Utilisation
const contentSearch = new ContentSearchEngine(db);
await contentSearch.initialize();

// Recherche avancée
const results = await contentSearch.search('MongoDB agrégations', {
  categories: ['Technical', 'Tutorial'],
  tags: ['database', 'nosql'],
  dateFrom: new Date('2024-01-01'),
  language: 'french',
  page: 1
});

// Articles similaires
const related = await contentSearch.searchRelated(articleId, 5);
console.log('Related articles:', related);

// Recherche contextuelle
const beginner = await contentSearch.searchByContext(
  'MongoDB introduction for beginners',
  10
);

// Highlights
const highlighted = await contentSearch.highlightMatches(
  articleId,
  'MongoDB agrégations'
);
console.log('Snippets:', highlighted.snippets);

// Résumé de recherche
const summary = await contentSearch.generateSearchSummary('MongoDB');
console.log('Search summary:', summary);
```

### Cas 3 : Recherche dans documentation technique

```javascript
class TechnicalDocSearch {
  constructor(db) {
    this.db = db;
    this.docs = null;
  }

  async initialize() {
    this.docs = this.db.collection('documentation');

    // Index text pour documentation
    try {
      await this.docs.createIndex(
        {
          title: "text",
          content: "text",
          "codeExamples.code": "text",
          "codeExamples.description": "text",
          apiSignature: "text",
          keywords: "text"
        },
        {
          weights: {
            title: 20,
            apiSignature: 15,
            keywords: 10,
            "codeExamples.description": 5,
            content: 3,
            "codeExamples.code": 2
          },
          name: "documentation_search"
        }
      );
    } catch (error) {
      if (error.code !== 85) throw error;
    }

    // Index pour navigation
    await this.docs.createIndex({ section: 1, version: -1 });
    await this.docs.createIndex({ type: 1 });
  }

  async search(query, options = {}) {
    const {
      sections = [],
      types = [],        // 'tutorial', 'reference', 'guide', 'api'
      version = 'latest',
      includeCode = true,
      page = 1,
      limit = 15
    } = options;

    const pipeline = [];

    // Recherche text
    pipeline.push({
      $match: {
        $text: { $search: query }
      }
    });

    // Score
    pipeline.push({
      $addFields: {
        searchScore: { $meta: "textScore" }
      }
    });

    // Filtres
    const filters = {};

    if (sections.length > 0) {
      filters.section = { $in: sections };
    }

    if (types.length > 0) {
      filters.type = { $in: types };
    }

    if (version !== 'all') {
      filters.$or = [
        { version },
        { version: 'all' }
      ];
    }

    pipeline.push({ $match: filters });

    // Détection type de query
    const queryType = this.detectQueryType(query);

    // Scoring adapté au type de query
    pipeline.push({
      $addFields: {
        relevanceScore: {
          $add: [
            { $multiply: ['$searchScore', this.getBaseMultiplier(queryType)] },
            {
              $cond: [
                { $eq: ['$type', queryType.preferredType] },
                5,
                0
              ]
            },
            {
              $cond: [
                { $in: ['$section', queryType.preferredSections || []] },
                3,
                0
              ]
            },
            { $divide: ['$pageViews', 1000] }
          ]
        }
      }
    });

    // Tri
    pipeline.push({ $sort: { relevanceScore: -1 } });

    // Pagination
    pipeline.push({ $skip: (page - 1) * limit });
    pipeline.push({ $limit: limit });

    // Projection
    const projection = {
      title: 1,
      summary: 1,
      section: 1,
      type: 1,
      url: 1,
      version: 1,
      searchScore: 1,
      relevanceScore: 1
    };

    if (includeCode) {
      projection.codeExamples = { $slice: ['$codeExamples', 2] };
    }

    pipeline.push({ $project: projection });

    const results = await this.docs.aggregate(pipeline).toArray();

    return {
      results,
      queryType: queryType.type,
      suggestions: this.generateSuggestions(query, queryType)
    };
  }

  detectQueryType(query) {
    const lowerQuery = query.toLowerCase();

    // Type: API Reference
    if (/\(|\)|\.|::|->/.test(query)) {
      return {
        type: 'api',
        preferredType: 'reference',
        preferredSections: ['API Reference'],
        baseMultiplier: 2
      };
    }

    // Type: How-to / Tutorial
    if (/how to|how do|tutorial|example|guide/i.test(query)) {
      return {
        type: 'howto',
        preferredType: 'tutorial',
        preferredSections: ['Tutorials', 'Guides'],
        baseMultiplier: 1.5
      };
    }

    // Type: Concept / Explanation
    if (/what is|explain|understand|concept/i.test(query)) {
      return {
        type: 'concept',
        preferredType: 'guide',
        preferredSections: ['Guides', 'Concepts'],
        baseMultiplier: 1.5
      };
    }

    // Type: Troubleshooting
    if (/error|fix|problem|issue|debug|troubleshoot/i.test(query)) {
      return {
        type: 'troubleshooting',
        preferredType: 'guide',
        preferredSections: ['Troubleshooting', 'Common Issues'],
        baseMultiplier: 1.5
      };
    }

    // Type: Generic
    return {
      type: 'generic',
      baseMultiplier: 1
    };
  }

  getBaseMultiplier(queryType) {
    return queryType.baseMultiplier || 1;
  }

  generateSuggestions(query, queryType) {
    const suggestions = [];

    switch (queryType.type) {
      case 'api':
        suggestions.push(`Browse ${query} API Reference`);
        suggestions.push(`${query} code examples`);
        break;

      case 'howto':
        suggestions.push(`${query} step by step`);
        suggestions.push(`${query} best practices`);
        break;

      case 'concept':
        suggestions.push(`${query} documentation`);
        suggestions.push(`${query} overview`);
        break;

      case 'troubleshooting':
        suggestions.push(`${query} solutions`);
        suggestions.push(`Common ${query} problems`);
        break;
    }

    return suggestions;
  }

  async searchCode(codeSnippet) {
    // Recherche spécifique dans exemples de code
    return await this.docs.aggregate([
      {
        $match: {
          $text: { $search: codeSnippet }
        }
      },
      {
        $addFields: {
          score: { $meta: "textScore" }
        }
      },
      { $unwind: '$codeExamples' },
      {
        $addFields: {
          codeMatch: {
            $regexMatch: {
              input: '$codeExamples.code',
              regex: codeSnippet,
              options: 'i'
            }
          }
        }
      },
      { $match: { codeMatch: true } },
      { $sort: { score: -1 } },
      { $limit: 10 },
      {
        $project: {
          title: 1,
          url: 1,
          codeExample: '$codeExamples',
          score: 1
        }
      }
    ]).toArray();
  }

  async searchSimilarDocs(docId) {
    const doc = await this.docs.findOne({ _id: docId });
    if (!doc) throw new Error('Document not found');

    const searchTerms = [
      doc.title,
      ...(doc.keywords || [])
    ].join(' ');

    return await this.docs.aggregate([
      {
        $match: {
          _id: { $ne: docId },
          $text: { $search: searchTerms }
        }
      },
      {
        $addFields: {
          textScore: { $meta: "textScore" },
          keywordOverlap: {
            $size: {
              $setIntersection: [
                doc.keywords || [],
                { $ifNull: ['$keywords', []] }
              ]
            }
          }
        }
      },
      {
        $addFields: {
          similarityScore: {
            $add: [
              '$textScore',
              { $multiply: ['$keywordOverlap', 3] }
            ]
          }
        }
      },
      { $sort: { similarityScore: -1 } },
      { $limit: 5 }
    ]).toArray();
  }

  async buildSearchIndex() {
    // Construire index inversé pour autocomplétion rapide
    const terms = await this.docs.aggregate([
      { $project: { title: 1, keywords: 1 } },
      {
        $project: {
          terms: {
            $concatArrays: [
              { $split: [{ $toLower: '$title' }, ' '] },
              { $map: {
                input: { $ifNull: ['$keywords', []] },
                as: 'kw',
                in: { $toLower: '$$kw' }
              }}
            ]
          }
        }
      },
      { $unwind: '$terms' },
      { $match: { terms: { $ne: '' } } },
      {
        $group: {
          _id: '$terms',
          frequency: { $sum: 1 },
          docIds: { $addToSet: '$_id' }
        }
      },
      { $match: { frequency: { $gte: 2 } } },
      { $sort: { frequency: -1 } }
    ]).toArray();

    // Sauvegarder dans collection dédiée
    await this.db.collection('search_index').deleteMany({});
    await this.db.collection('search_index').insertMany(terms);

    return terms.length;
  }

  async autocomplete(prefix, limit = 10) {
    const regex = new RegExp(`^${prefix}`, 'i');

    return await this.db.collection('search_index').find({
      _id: regex
    })
      .sort({ frequency: -1 })
      .limit(limit)
      .project({ term: '$_id', frequency: 1, _id: 0 })
      .toArray();
  }
}

// Utilisation
const docSearch = new TechnicalDocSearch(db);
await docSearch.initialize();

// Recherche générale
const results = await docSearch.search('aggregation pipeline', {
  types: ['tutorial', 'reference'],
  version: '7.0',
  includeCode: true
});

console.log('Query type:', results.queryType);
console.log('Results:', results.results);
console.log('Suggestions:', results.suggestions);

// Recherche de code
const codeResults = await docSearch.searchCode('db.collection.aggregate');
console.log('Code examples:', codeResults);

// Documents similaires
const similar = await docSearch.searchSimilarDocs(docId);

// Build search index
const indexSize = await docSearch.buildSearchIndex();
console.log(`Search index built: ${indexSize} terms`);

// Autocomplétion
const completions = await docSearch.autocomplete('aggr');
console.log('Autocomplete:', completions);
```

---

## Performance et optimisations

### Limites et contraintes

```javascript
// ⚠️ Une seule recherche $text par query
// ❌ INVALIDE
db.collection.find({
  $text: { $search: "mongodb" },
  $text: { $search: "database" }  // Erreur!
});

// ✅ VALIDE - Combiner termes
db.collection.find({
  $text: { $search: "mongodb database" }
});

// ⚠️ $text doit être premier stage en aggregation
// ❌ INVALIDE
db.collection.aggregate([
  { $match: { category: 'tech' } },
  { $match: { $text: { $search: "mongodb" } } }  // Erreur!
]);

// ✅ VALIDE - $text en premier
db.collection.aggregate([
  { $match: { $text: { $search: "mongodb" } } },
  { $match: { category: 'tech' } }
]);

// ⚠️ Un seul index text par collection
// Créer index composé si besoin de plusieurs champs
```

### Optimisation des index

```javascript
// ✅ Index text avec champs de filtre fréquents
await collection.createIndex({
  // Text fields
  title: "text",
  content: "text",
  // Regular fields pour filtres
  category: 1,
  status: 1,
  publishedAt: -1
});

// Query optimisée
const results = await collection.find({
  $text: { $search: "mongodb" },
  category: "Database",  // Utilise l'index
  status: "published"     // Utilise l'index
}).toArray();
```

### Caching de résultats

```javascript
class SearchResultCache {
  constructor(redisClient) {
    this.redis = redisClient;
    this.ttl = 3600;  // 1 heure
  }

  generateKey(query, filters = {}) {
    const filterStr = JSON.stringify(filters);
    return `search:${query.toLowerCase()}:${filterStr}`;
  }

  async get(query, filters) {
    const key = this.generateKey(query, filters);
    const cached = await this.redis.get(key);

    if (cached) {
      return {
        cached: true,
        data: JSON.parse(cached),
        source: 'cache'
      };
    }

    return { cached: false };
  }

  async set(query, filters, results) {
    const key = this.generateKey(query, filters);
    await this.redis.setex(
      key,
      this.ttl,
      JSON.stringify(results)
    );
  }

  async invalidate(pattern) {
    const keys = await this.redis.keys(`search:${pattern}*`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
}

// Utilisation
const cache = new SearchResultCache(redisClient);

async function search(query, filters) {
  // Check cache
  const cached = await cache.get(query, filters);
  if (cached.cached) {
    return cached.data;
  }

  // Query database
  const results = await db.collection('products').find({
    $text: { $search: query },
    ...filters
  }).toArray();

  // Cache results
  await cache.set(query, filters, results);

  return results;
}
```

---

## MongoDB Atlas Search

Pour des besoins avancés, MongoDB Atlas Search offre des capacités supérieures :

```javascript
// Atlas Search (nécessite Atlas)
db.collection('products').aggregate([
  {
    $search: {
      index: "products_search",
      text: {
        query: "laptop gaming",
        path: ["name", "description"],
        fuzzy: {
          maxEdits: 2,
          prefixLength: 3
        }
      },
      highlight: {
        path: ["name", "description"]
      }
    }
  },
  {
    $project: {
      name: 1,
      description: 1,
      score: { $meta: "searchScore" },
      highlights: { $meta: "searchHighlights" }
    }
  }
]).toArray();

// Autocomplete avec Atlas Search
db.collection('products').aggregate([
  {
    $search: {
      index: "products_autocomplete",
      autocomplete: {
        query: "lapt",
        path: "name",
        fuzzy: {
          maxEdits: 1
        }
      }
    }
  }
]).toArray();
```

### Comparaison Text Index vs Atlas Search

| Fonctionnalité | Text Index | Atlas Search |
|----------------|------------|--------------|
| **Setup** | Intégré MongoDB | Nécessite Atlas |
| **Performance** | Bonne | Excellente |
| **Fuzzy search** | ❌ Non | ✅ Oui |
| **Autocomplete** | Basique (regex) | ✅ Optimisé |
| **Facets** | Manuel (aggregation) | ✅ Intégré |
| **Synonymes** | ❌ Non | ✅ Oui |
| **Highlighting** | Manuel | ✅ Automatique |
| **Typo tolerance** | ❌ Non | ✅ Oui |
| **Coût** | Inclus | Atlas uniquement |

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Créer index text avec weights appropriés
await collection.createIndex(
  { title: "text", content: "text" },
  { weights: { title: 10, content: 1 } }
);

// 2. Utiliser projection pour limiter données retournées
const results = await collection.find(
  { $text: { $search: query } },
  { score: { $meta: "textScore" }, title: 1, excerpt: 1 }
).toArray();

// 3. Limiter résultats
const limited = await collection.find(
  { $text: { $search: query } }
).limit(50).toArray();

// 4. Combiner avec autres filtres
const filtered = await collection.find({
  $text: { $search: query },
  category: "Tech",
  status: "published"
}).toArray();

// 5. Tracker analytics de recherche
await logSearch(query, results.length, userId);
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas créer index text sur champs énormes
// ❌ Performance impact
await collection.createIndex({
  massiveTextField: "text"  // Plusieurs Mo de texte
});

// 2. Ne pas faire recherche sans limite
// ❌ Peut retourner des millions de résultats
const unlimited = await collection.find({
  $text: { $search: query }
}).toArray();

// 3. Ne pas oublier le tri par score
// ❌ Résultats non pertinents en premier
const unsorted = await collection.find({
  $text: { $search: query }
}).toArray();

// ✅ Trier par pertinence
const sorted = await collection.find(
  { $text: { $search: query } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } }).toArray();

// 4. Ne pas utiliser regex pour recherche fulltext
// ❌ Très lent, pas de stemming
const regex = await collection.find({
  content: /mongodb/i
}).toArray();

// 5. Ne pas négliger la langue
// ❌ Stemming incorrect
await collection.createIndex(
  { content: "text" }
  // Manque default_language
);
```

---

## Conclusion

La recherche Full-Text de MongoDB offre :
- ✅ **Support multilingue** (15+ langues avec stemming)
- ✅ **Scoring de pertinence** automatique
- ✅ **Pondération flexible** des champs
- ✅ **Intégration native** avec agrégations
- ✅ **Performance correcte** pour besoins standards

**Points clés à retenir :**
1. Un seul index text par collection (mais composé multi-champs)
2. Weights pour prioriser certains champs
3. Langue importante pour stemming correct
4. Score de pertinence pour tri
5. Combiner avec filtres standards pour précision
6. Atlas Search pour besoins avancés (fuzzy, autocomplete, etc.)

**Cas d'usage idéaux :**
- Moteurs de recherche e-commerce
- Recherche de contenu/blog
- Documentation technique
- Bases de connaissances
- Catalogues de produits

**Quand utiliser Atlas Search :**
- Recherche floue (typo tolerance)
- Autocomplétion optimisée
- Synonymes
- Highlighting automatique
- Performance critique sur gros volumes

---


⏭️ [Atlas Search et Lucene](/16-fonctionnalites-avancees/08-atlas-search-lucene.md)
