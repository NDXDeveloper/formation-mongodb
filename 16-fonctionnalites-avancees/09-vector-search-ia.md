🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.9 Vector Search et IA

## Introduction

La **recherche vectorielle** (Vector Search) est une technique qui permet de rechercher des données basées sur leur similarité sémantique plutôt que sur des mots-clés exacts. Combinée à l'intelligence artificielle, elle permet de construire des applications sophistiquées : moteurs de recherche sémantique, systèmes de recommandation, chatbots avec contexte (RAG), et bien plus.

MongoDB Atlas Vector Search utilise des **embeddings** (représentations numériques de données) pour capturer le sens et le contexte, permettant de trouver des contenus similaires même avec des formulations différentes.

---

## Concepts fondamentaux

### Qu'est-ce qu'un embedding ?

```javascript
// Texte original
const text = "MongoDB est une base de données NoSQL";

// Embedding (vecteur de 768 dimensions)
const embedding = [
  0.023, -0.145, 0.892, ..., -0.334  // 768 valeurs
];

// Propriétés des embeddings:
// - Textes similaires = vecteurs proches
// - Distance calculable (cosine, euclidean, dot product)
// - Capture le sens sémantique, pas seulement les mots
```

### Similarité vectorielle

```javascript
// Mesures de similarité courantes
const similarities = {
  cosine: "Angle entre vecteurs (0 à 1, 1 = identique)",
  euclidean: "Distance euclidienne (0 = identique, plus grand = différent)",
  dotProduct: "Produit scalaire (plus grand = plus similaire)"
};

// Exemple: Cosine similarity
function cosineSimilarity(vec1, vec2) {
  const dotProduct = vec1.reduce((sum, val, i) => sum + val * vec2[i], 0);
  const mag1 = Math.sqrt(vec1.reduce((sum, val) => sum + val * val, 0));
  const mag2 = Math.sqrt(vec2.reduce((sum, val) => sum + val * val, 0));
  return dotProduct / (mag1 * mag2);
}

// Similarité entre "MongoDB" et "database" : ~0.85
// Similarité entre "MongoDB" et "banana" : ~0.12
```

---

## Configuration Vector Search dans Atlas

### Création d'index vectoriel

```javascript
// Configuration via Atlas UI ou API
const vectorIndexConfig = {
  name: "vector_index",
  type: "vectorSearch",
  definition: {
    fields: [
      {
        type: "vector",
        path: "embedding",
        numDimensions: 1536,  // Dimension des embeddings (ex: OpenAI ada-002)
        similarity: "cosine"   // ou "euclidean", "dotProduct"
      },
      {
        type: "filter",
        path: "category"  // Champs pour pre-filtering
      },
      {
        type: "filter",
        path: "tags"
      }
    ]
  }
};

// Index avec multiple vecteurs
const multiVectorIndexConfig = {
  name: "multi_vector_index",
  definition: {
    fields: [
      {
        type: "vector",
        path: "titleEmbedding",
        numDimensions: 768,
        similarity: "cosine"
      },
      {
        type: "vector",
        path: "contentEmbedding",
        numDimensions: 768,
        similarity: "cosine"
      }
    ]
  }
};
```

### Dimensions communes des embeddings

```javascript
const embeddingModels = {
  // OpenAI
  "text-embedding-ada-002": 1536,
  "text-embedding-3-small": 1536,
  "text-embedding-3-large": 3072,

  // Cohere
  "embed-english-v3.0": 1024,
  "embed-multilingual-v3.0": 1024,

  // Sentence Transformers
  "all-MiniLM-L6-v2": 384,
  "all-mpnet-base-v2": 768,

  // Custom models
  "bert-base-uncased": 768,
  "roberta-large": 1024
};
```

---

## Génération d'embeddings

### Intégration avec OpenAI

```javascript
const { OpenAI } = require('openai');

class EmbeddingGenerator {
  constructor(apiKey) {
    this.openai = new OpenAI({ apiKey });
    this.model = "text-embedding-3-small";
    this.dimensions = 1536;
  }

  async generateEmbedding(text) {
    try {
      const response = await this.openai.embeddings.create({
        model: this.model,
        input: text,
        encoding_format: "float"
      });

      return response.data[0].embedding;
    } catch (error) {
      console.error('Embedding generation failed:', error);
      throw error;
    }
  }

  async generateEmbeddings(texts) {
    // Batch processing pour efficacité
    const batchSize = 100;
    const embeddings = [];

    for (let i = 0; i < texts.length; i += batchSize) {
      const batch = texts.slice(i, i + batchSize);

      const response = await this.openai.embeddings.create({
        model: this.model,
        input: batch
      });

      embeddings.push(...response.data.map(d => d.embedding));
    }

    return embeddings;
  }

  async estimateCost(textCount, avgTokens = 500) {
    // OpenAI pricing estimation
    const costPer1MTokens = 0.00002; // text-embedding-3-small
    const totalTokens = textCount * avgTokens;
    return (totalTokens / 1000000) * costPer1MTokens;
  }
}

// Utilisation
const embedder = new EmbeddingGenerator(process.env.OPENAI_API_KEY);

const text = "MongoDB est une base de données NoSQL orientée documents";
const embedding = await embedder.generateEmbedding(text);

console.log('Embedding dimensions:', embedding.length);
console.log('First 5 values:', embedding.slice(0, 5));
```

### Intégration avec Cohere

```javascript
const { CohereClient } = require('cohere-ai');

class CohereEmbedder {
  constructor(apiKey) {
    this.cohere = new CohereClient({ token: apiKey });
    this.model = "embed-multilingual-v3.0";
  }

  async generateEmbedding(text, inputType = "search_document") {
    // inputType: "search_document", "search_query", "classification", "clustering"
    const response = await this.cohere.embed({
      texts: [text],
      model: this.model,
      inputType: inputType,
      embeddingTypes: ["float"]
    });

    return response.embeddings.float[0];
  }

  async generateEmbeddingsWithContext(texts, inputType) {
    const response = await this.cohere.embed({
      texts,
      model: this.model,
      inputType,
      truncate: "END"
    });

    return response.embeddings.float;
  }
}
```

### Embeddings locaux (Sentence Transformers)

```javascript
// Pour éviter les coûts API, utiliser modèle local
const { pipeline } = require('@xenova/transformers');

class LocalEmbedder {
  constructor() {
    this.model = null;
    this.modelName = 'Xenova/all-MiniLM-L6-v2';
  }

  async initialize() {
    this.model = await pipeline(
      'feature-extraction',
      this.modelName
    );
  }

  async generateEmbedding(text) {
    if (!this.model) await this.initialize();

    const output = await this.model(text, {
      pooling: 'mean',
      normalize: true
    });

    return Array.from(output.data);
  }

  async generateEmbeddingsBatch(texts) {
    if (!this.model) await this.initialize();

    const embeddings = await Promise.all(
      texts.map(text => this.generateEmbedding(text))
    );

    return embeddings;
  }
}

// Utilisation
const localEmbedder = new LocalEmbedder();
await localEmbedder.initialize();

const embedding = await localEmbedder.generateEmbedding("MongoDB database");
console.log('Local embedding:', embedding.length);
```

---

## Cas d'usage avancés

### Cas 1 : RAG (Retrieval-Augmented Generation)

```javascript
class RAGSystem {
  constructor(db, openaiKey) {
    this.db = db;
    this.knowledge = db.collection('knowledge_base');
    this.embedder = new EmbeddingGenerator(openaiKey);
    this.openai = new OpenAI({ apiKey: openaiKey });
  }

  async setupVectorIndex() {
    // Configuration via Atlas
    const indexConfig = {
      name: "knowledge_vector_index",
      definition: {
        fields: [
          {
            type: "vector",
            path: "embedding",
            numDimensions: 1536,
            similarity: "cosine"
          },
          {
            type: "filter",
            path: "category"
          },
          {
            type: "filter",
            path: "source"
          }
        ]
      }
    };

    console.log('Apply this config via Atlas UI');
  }

  async indexDocument(document) {
    // Préparer texte pour embedding
    const textToEmbed = this.prepareTextForEmbedding(document);

    // Générer embedding
    const embedding = await this.embedder.generateEmbedding(textToEmbed);

    // Stocker avec embedding
    await this.knowledge.insertOne({
      ...document,
      embedding,
      textForSearch: textToEmbed,
      indexedAt: new Date()
    });

    return embedding;
  }

  prepareTextForEmbedding(document) {
    // Combiner champs pertinents
    return [
      document.title,
      document.summary,
      document.content
    ].filter(Boolean).join('\n\n');
  }

  async indexDocumentsBatch(documents, batchSize = 50) {
    for (let i = 0; i < documents.length; i += batchSize) {
      const batch = documents.slice(i, i + batchSize);

      // Préparer textes
      const texts = batch.map(doc => this.prepareTextForEmbedding(doc));

      // Générer embeddings en batch
      const embeddings = await this.embedder.generateEmbeddings(texts);

      // Préparer documents avec embeddings
      const docsWithEmbeddings = batch.map((doc, index) => ({
        ...doc,
        embedding: embeddings[index],
        textForSearch: texts[index],
        indexedAt: new Date()
      }));

      // Insérer batch
      await this.knowledge.insertMany(docsWithEmbeddings);

      console.log(`Indexed ${i + batch.length}/${documents.length} documents`);
    }
  }

  async retrieveRelevantContext(query, options = {}) {
    const {
      limit = 5,
      minScore = 0.7,
      category = null,
      sources = []
    } = options;

    // Générer embedding de la query
    const queryEmbedding = await this.embedder.generateEmbedding(query);

    // Pipeline de recherche vectorielle
    const pipeline = [
      {
        $vectorSearch: {
          index: "knowledge_vector_index",
          path: "embedding",
          queryVector: queryEmbedding,
          numCandidates: limit * 10,
          limit: limit
        }
      }
    ];

    // Filtres optionnels
    if (category || sources.length > 0) {
      const matchStage = {};
      if (category) matchStage.category = category;
      if (sources.length > 0) matchStage.source = { $in: sources };

      pipeline.push({ $match: matchStage });
    }

    // Ajouter score
    pipeline.push({
      $addFields: {
        searchScore: { $meta: "vectorSearchScore" }
      }
    });

    // Filtrer par score minimum
    pipeline.push({
      $match: {
        searchScore: { $gte: minScore }
      }
    });

    // Projection
    pipeline.push({
      $project: {
        title: 1,
        content: 1,
        summary: 1,
        source: 1,
        category: 1,
        searchScore: 1,
        embedding: 0  // Ne pas retourner l'embedding
      }
    });

    return await this.knowledge.aggregate(pipeline).toArray();
  }

  async generateAnswer(question, options = {}) {
    const {
      maxContextLength = 4000,
      temperature = 0.7,
      model = "gpt-4-turbo-preview"
    } = options;

    // 1. Récupérer contexte pertinent
    console.log('Retrieving relevant context...');
    const relevantDocs = await this.retrieveRelevantContext(question, {
      limit: 5,
      minScore: 0.75,
      ...options
    });

    if (relevantDocs.length === 0) {
      return {
        answer: "Je n'ai pas trouvé d'informations pertinentes pour répondre à cette question.",
        confidence: 0,
        sources: []
      };
    }

    // 2. Construire contexte
    const context = this.buildContext(relevantDocs, maxContextLength);

    // 3. Construire prompt
    const prompt = this.buildPrompt(question, context);

    // 4. Générer réponse avec LLM
    console.log('Generating answer...');
    const completion = await this.openai.chat.completions.create({
      model,
      messages: [
        {
          role: "system",
          content: "Tu es un assistant qui répond aux questions en te basant uniquement sur le contexte fourni. Si le contexte ne contient pas l'information nécessaire, dis-le clairement."
        },
        {
          role: "user",
          content: prompt
        }
      ],
      temperature,
      max_tokens: 1000
    });

    const answer = completion.choices[0].message.content;

    // 5. Calculer confiance basée sur scores
    const avgScore = relevantDocs.reduce((sum, doc) => sum + doc.searchScore, 0)
                     / relevantDocs.length;

    return {
      answer,
      confidence: avgScore,
      sources: relevantDocs.map(doc => ({
        title: doc.title,
        source: doc.source,
        score: doc.searchScore
      })),
      contextUsed: context.length
    };
  }

  buildContext(documents, maxLength) {
    let context = "";

    for (const doc of documents) {
      const docText = `
Source: ${doc.title} (${doc.source})
Score: ${doc.searchScore.toFixed(3)}

${doc.content}

---
`;

      if ((context + docText).length > maxLength) {
        break;
      }

      context += docText;
    }

    return context;
  }

  buildPrompt(question, context) {
    return `Contexte:
${context}

Question: ${question}

Réponds à la question en te basant uniquement sur le contexte ci-dessus. Si le contexte ne contient pas assez d'informations, indique-le clairement. Cite tes sources.`;
  }

  async conversationalRAG(conversationHistory, newQuestion) {
    // RAG avec historique de conversation

    // 1. Générer query embedding incluant contexte conversation
    const contextualQuery = this.buildContextualQuery(
      conversationHistory,
      newQuestion
    );

    const queryEmbedding = await this.embedder.generateEmbedding(contextualQuery);

    // 2. Recherche vectorielle
    const relevantDocs = await this.knowledge.aggregate([
      {
        $vectorSearch: {
          index: "knowledge_vector_index",
          path: "embedding",
          queryVector: queryEmbedding,
          numCandidates: 50,
          limit: 5
        }
      },
      {
        $addFields: {
          score: { $meta: "vectorSearchScore" }
        }
      }
    ]).toArray();

    // 3. Construire prompt avec historique
    const messages = [
      {
        role: "system",
        content: "Tu es un assistant conversationnel. Réponds en tenant compte de l'historique de la conversation et du contexte fourni."
      },
      ...conversationHistory,
      {
        role: "user",
        content: `Contexte pertinent:
${this.buildContext(relevantDocs, 3000)}

Question: ${newQuestion}`
      }
    ];

    // 4. Générer réponse
    const completion = await this.openai.chat.completions.create({
      model: "gpt-4-turbo-preview",
      messages,
      temperature: 0.7
    });

    return {
      answer: completion.choices[0].message.content,
      sources: relevantDocs.map(d => ({
        title: d.title,
        score: d.score
      }))
    };
  }

  buildContextualQuery(history, newQuestion) {
    // Combiner derniers messages + nouvelle question
    const recentMessages = history.slice(-3);
    const contextText = recentMessages
      .map(m => m.content)
      .join(' ');

    return `${contextText} ${newQuestion}`;
  }

  async evaluateAnswer(question, generatedAnswer, groundTruth) {
    // Évaluer qualité de la réponse (pour tests)
    const prompt = `Question: ${question}

Réponse générée: ${generatedAnswer}

Réponse attendue: ${groundTruth}

Évalue la qualité de la réponse générée sur une échelle de 1 à 10 en considérant:
- Exactitude factuelle
- Complétude
- Pertinence
- Clarté

Réponds uniquement avec un nombre de 1 à 10.`;

    const evaluation = await this.openai.chat.completions.create({
      model: "gpt-4",
      messages: [{ role: "user", content: prompt }],
      temperature: 0
    });

    const score = parseInt(evaluation.choices[0].message.content);
    return { score, isValid: score >= 7 };
  }
}

// Utilisation
const rag = new RAGSystem(db, process.env.OPENAI_API_KEY);
await rag.setupVectorIndex();

// Indexer base de connaissances
const documents = [
  {
    title: "Introduction à MongoDB",
    content: "MongoDB est une base de données NoSQL orientée documents...",
    category: "database",
    source: "documentation"
  },
  // ... plus de documents
];

await rag.indexDocumentsBatch(documents);

// Poser une question
const result = await rag.generateAnswer(
  "Comment fonctionne la réplication dans MongoDB?"
);

console.log('Answer:', result.answer);
console.log('Confidence:', result.confidence);
console.log('Sources:', result.sources);

// RAG conversationnel
const conversation = [
  { role: "user", content: "Qu'est-ce que MongoDB?" },
  { role: "assistant", content: "MongoDB est une base de données NoSQL..." },
];

const response = await rag.conversationalRAG(
  conversation,
  "Comment puis-je l'installer?"
);
```

### Cas 2 : Recherche sémantique multimodale

```javascript
class MultimodalSemanticSearch {
  constructor(db, openaiKey) {
    this.db = db;
    this.documents = db.collection('documents');
    this.embedder = new EmbeddingGenerator(openaiKey);
    this.openai = new OpenAI({ apiKey: openaiKey });
  }

  async setupIndexes() {
    // Index avec embeddings multiples
    const indexConfig = {
      name: "multimodal_index",
      definition: {
        fields: [
          {
            type: "vector",
            path: "titleEmbedding",
            numDimensions: 1536,
            similarity: "cosine"
          },
          {
            type: "vector",
            path: "contentEmbedding",
            numDimensions: 1536,
            similarity: "cosine"
          },
          {
            type: "vector",
            path: "summaryEmbedding",
            numDimensions: 1536,
            similarity: "cosine"
          },
          {
            type: "filter",
            path: "documentType"
          },
          {
            type: "filter",
            path: "language"
          }
        ]
      }
    };

    console.log('Configure via Atlas UI');
  }

  async indexDocument(document) {
    // Générer embeddings pour différentes parties
    const [titleEmb, contentEmb, summaryEmb] = await Promise.all([
      this.embedder.generateEmbedding(document.title),
      this.embedder.generateEmbedding(document.content),
      this.embedder.generateEmbedding(document.summary || document.content.slice(0, 500))
    ]);

    await this.documents.insertOne({
      ...document,
      titleEmbedding: titleEmb,
      contentEmbedding: contentEmb,
      summaryEmbedding: summaryEmb,
      indexedAt: new Date()
    });
  }

  async semanticSearch(query, options = {}) {
    const {
      searchMode = 'balanced',  // 'title', 'content', 'balanced'
      limit = 20,
      filters = {}
    } = options;

    // Générer embedding de la query
    const queryEmbedding = await this.embedder.generateEmbedding(query);

    // Sélectionner champ d'embedding selon mode
    let embeddingPath;
    let weights;

    switch (searchMode) {
      case 'title':
        embeddingPath = 'titleEmbedding';
        break;
      case 'content':
        embeddingPath = 'contentEmbedding';
        break;
      case 'balanced':
        // Recherche sur titre et contenu
        return await this.balancedSearch(queryEmbedding, limit, filters);
      default:
        embeddingPath = 'contentEmbedding';
    }

    // Pipeline de recherche
    const pipeline = [
      {
        $vectorSearch: {
          index: "multimodal_index",
          path: embeddingPath,
          queryVector: queryEmbedding,
          numCandidates: limit * 5,
          limit: limit,
          filter: this.buildFilters(filters)
        }
      },
      {
        $addFields: {
          searchScore: { $meta: "vectorSearchScore" }
        }
      },
      {
        $project: {
          title: 1,
          summary: 1,
          documentType: 1,
          author: 1,
          publishedAt: 1,
          searchScore: 1,
          // Ne pas retourner les embeddings
          titleEmbedding: 0,
          contentEmbedding: 0,
          summaryEmbedding: 0
        }
      }
    ];

    return await this.documents.aggregate(pipeline).toArray();
  }

  async balancedSearch(queryEmbedding, limit, filters) {
    // Recherche pondérée sur titre (60%) et contenu (40%)
    const titleWeight = 0.6;
    const contentWeight = 0.4;

    // Recherche parallèle
    const [titleResults, contentResults] = await Promise.all([
      this.documents.aggregate([
        {
          $vectorSearch: {
            index: "multimodal_index",
            path: "titleEmbedding",
            queryVector: queryEmbedding,
            numCandidates: limit * 3,
            limit: limit * 2,
            filter: this.buildFilters(filters)
          }
        },
        {
          $addFields: {
            titleScore: { $meta: "vectorSearchScore" }
          }
        }
      ]).toArray(),

      this.documents.aggregate([
        {
          $vectorSearch: {
            index: "multimodal_index",
            path: "contentEmbedding",
            queryVector: queryEmbedding,
            numCandidates: limit * 3,
            limit: limit * 2,
            filter: this.buildFilters(filters)
          }
        },
        {
          $addFields: {
            contentScore: { $meta: "vectorSearchScore" }
          }
        }
      ]).toArray()
    ]);

    // Fusionner et ré-scorer
    const scoreMap = new Map();

    titleResults.forEach(doc => {
      scoreMap.set(doc._id.toString(), {
        doc,
        titleScore: doc.titleScore || 0,
        contentScore: 0
      });
    });

    contentResults.forEach(doc => {
      const existing = scoreMap.get(doc._id.toString());
      if (existing) {
        existing.contentScore = doc.contentScore || 0;
      } else {
        scoreMap.set(doc._id.toString(), {
          doc,
          titleScore: 0,
          contentScore: doc.contentScore || 0
        });
      }
    });

    // Calculer score combiné
    const combined = Array.from(scoreMap.values()).map(item => ({
      ...item.doc,
      combinedScore: (
        item.titleScore * titleWeight +
        item.contentScore * contentWeight
      )
    }));

    // Trier et limiter
    return combined
      .sort((a, b) => b.combinedScore - a.combinedScore)
      .slice(0, limit);
  }

  buildFilters(filters) {
    const mongoFilters = {};

    if (filters.documentType) {
      mongoFilters.documentType = { $eq: filters.documentType };
    }

    if (filters.language) {
      mongoFilters.language = { $eq: filters.language };
    }

    if (filters.publishedAfter) {
      mongoFilters.publishedAt = { $gte: filters.publishedAfter };
    }

    return Object.keys(mongoFilters).length > 0 ? mongoFilters : undefined;
  }

  async crossLingualSearch(query, targetLanguages = []) {
    // Recherche cross-linguale (embedding capture le sens, pas la langue)
    const queryEmbedding = await this.embedder.generateEmbedding(query);

    const filters = targetLanguages.length > 0
      ? { language: { $in: targetLanguages } }
      : {};

    return await this.documents.aggregate([
      {
        $vectorSearch: {
          index: "multimodal_index",
          path: "contentEmbedding",
          queryVector: queryEmbedding,
          numCandidates: 100,
          limit: 20,
          filter: Object.keys(filters).length > 0 ? filters : undefined
        }
      },
      {
        $addFields: {
          searchScore: { $meta: "vectorSearchScore" }
        }
      },
      {
        $project: {
          title: 1,
          summary: 1,
          language: 1,
          searchScore: 1
        }
      }
    ]).toArray();
  }

  async queryExpansion(originalQuery) {
    // Expansion de requête avec LLM
    const prompt = `Given this search query: "${originalQuery}"

Generate 3 alternative phrasings that capture the same intent but use different words.

Respond with JSON:
{
  "expansions": ["query1", "query2", "query3"]
}`;

    const response = await this.openai.chat.completions.create({
      model: "gpt-3.5-turbo",
      messages: [{ role: "user", content: prompt }],
      response_format: { type: "json_object" }
    });

    const { expansions } = JSON.parse(response.choices[0].message.content);

    // Recherche avec queries expanded
    const allQueries = [originalQuery, ...expansions];

    const results = await Promise.all(
      allQueries.map(q => this.semanticSearch(q, { limit: 10 }))
    );

    // Dédupliquer et re-scorer
    const scoreMap = new Map();

    results.flat().forEach(doc => {
      const id = doc._id.toString();
      if (scoreMap.has(id)) {
        scoreMap.get(id).score = Math.max(scoreMap.get(id).score, doc.searchScore);
        scoreMap.get(id).appearances++;
      } else {
        scoreMap.set(id, {
          doc,
          score: doc.searchScore,
          appearances: 1
        });
      }
    });

    // Bonus pour documents apparaissant dans plusieurs expansions
    return Array.from(scoreMap.values())
      .map(item => ({
        ...item.doc,
        finalScore: item.score * (1 + 0.1 * item.appearances)
      }))
      .sort((a, b) => b.finalScore - a.finalScore)
      .slice(0, 20);
  }

  async semanticClustering(limit = 1000) {
    // Clustering sémantique de documents
    const docs = await this.documents.find({
      contentEmbedding: { $exists: true }
    })
      .limit(limit)
      .toArray();

    // K-means clustering (simplification)
    const embeddings = docs.map(d => d.contentEmbedding);
    const k = 10;  // 10 clusters

    // En production, utiliser bibliothèque ML appropriée
    const clusters = this.performKMeans(embeddings, k);

    return {
      clusters: clusters.map((clusterDocs, i) => ({
        clusterId: i,
        documentCount: clusterDocs.length,
        documents: clusterDocs.map(idx => ({
          title: docs[idx].title,
          summary: docs[idx].summary
        }))
      }))
    };
  }

  performKMeans(embeddings, k) {
    // Simplification - en production utiliser ML.js ou similar
    return Array.from({ length: k }, (_, i) =>
      embeddings
        .map((_, idx) => idx)
        .filter((_, idx) => idx % k === i)
    );
  }
}

// Utilisation
const semanticSearch = new MultimodalSemanticSearch(db, process.env.OPENAI_API_KEY);
await semanticSearch.setupIndexes();

// Recherche sémantique balanced
const results = await semanticSearch.semanticSearch(
  "machine learning databases",
  {
    searchMode: 'balanced',
    filters: {
      documentType: 'article',
      publishedAfter: new Date('2023-01-01')
    }
  }
);

// Recherche cross-linguale
const multiLang = await semanticSearch.crossLingualSearch(
  "artificial intelligence",
  ['en', 'fr', 'es']
);

// Query expansion
const expanded = await semanticSearch.queryExpansion(
  "NoSQL databases"
);

console.log('Expanded results:', expanded);
```

### Cas 3 : Système de recommandation intelligent

```javascript
class IntelligentRecommendationEngine {
  constructor(db, openaiKey) {
    this.db = db;
    this.items = db.collection('items');
    this.users = db.collection('users');
    this.interactions = db.collection('user_interactions');
    this.embedder = new EmbeddingGenerator(openaiKey);
  }

  async setupVectorIndexes() {
    // Index pour items
    const itemIndexConfig = {
      name: "item_vector_index",
      definition: {
        fields: [
          {
            type: "vector",
            path: "embedding",
            numDimensions: 1536,
            similarity: "cosine"
          },
          {
            type: "filter",
            path: "category"
          },
          {
            type: "filter",
            path: "tags"
          }
        ]
      }
    };

    // Index pour profils utilisateurs
    const userIndexConfig = {
      name: "user_vector_index",
      definition: {
        fields: [
          {
            type: "vector",
            path: "preferenceEmbedding",
            numDimensions: 1536,
            similarity: "cosine"
          }
        ]
      }
    };

    console.log('Configure both indexes via Atlas UI');
  }

  async indexItem(item) {
    // Créer description riche pour embedding
    const description = [
      item.name,
      item.description,
      item.category,
      ...item.tags
    ].join(' ');

    const embedding = await this.embedder.generateEmbedding(description);

    await this.items.updateOne(
      { _id: item._id },
      {
        $set: {
          embedding,
          embeddingDescription: description,
          lastEmbeddingUpdate: new Date()
        }
      },
      { upsert: true }
    );

    return embedding;
  }

  async buildUserProfile(userId) {
    // Construire profil utilisateur basé sur historique
    const interactions = await this.interactions.find({
      userId,
      type: { $in: ['view', 'like', 'purchase'] }
    })
      .sort({ timestamp: -1 })
      .limit(50)
      .toArray();

    if (interactions.length === 0) {
      return null;
    }

    // Récupérer items interactés
    const itemIds = interactions.map(i => i.itemId);
    const items = await this.items.find({
      _id: { $in: itemIds },
      embedding: { $exists: true }
    }).toArray();

    // Calculer embedding moyen pondéré
    const weights = {
      purchase: 3.0,
      like: 2.0,
      view: 1.0
    };

    let weightedSum = new Array(1536).fill(0);
    let totalWeight = 0;

    interactions.forEach(interaction => {
      const item = items.find(i => i._id.equals(interaction.itemId));
      if (!item || !item.embedding) return;

      const weight = weights[interaction.type] || 1.0;

      item.embedding.forEach((val, idx) => {
        weightedSum[idx] += val * weight;
      });

      totalWeight += weight;
    });

    const preferenceEmbedding = weightedSum.map(val => val / totalWeight);

    // Sauvegarder profil
    await this.users.updateOne(
      { _id: userId },
      {
        $set: {
          preferenceEmbedding,
          interactionCount: interactions.length,
          lastProfileUpdate: new Date()
        }
      },
      { upsert: true }
    );

    return preferenceEmbedding;
  }

  async recommendForUser(userId, options = {}) {
    const {
      limit = 10,
      excludeInteracted = true,
      categories = [],
      diversityBoost = 0.2
    } = options;

    // Récupérer profil utilisateur
    const user = await this.users.findOne({ _id: userId });

    if (!user || !user.preferenceEmbedding) {
      // Fallback: items populaires
      return await this.getPopularItems(limit);
    }

    // Items déjà vus/achetés
    let excludedIds = [];
    if (excludeInteracted) {
      const interactions = await this.interactions.find({
        userId,
        type: { $in: ['purchase', 'like'] }
      }).toArray();

      excludedIds = interactions.map(i => i.itemId);
    }

    // Recherche vectorielle
    const pipeline = [
      {
        $vectorSearch: {
          index: "item_vector_index",
          path: "embedding",
          queryVector: user.preferenceEmbedding,
          numCandidates: limit * 10,
          limit: limit * 3  // Chercher plus pour diversité
        }
      }
    ];

    // Exclure items déjà interactés
    if (excludedIds.length > 0) {
      pipeline.push({
        $match: {
          _id: { $nin: excludedIds }
        }
      });
    }

    // Filtrer par catégories
    if (categories.length > 0) {
      pipeline.push({
        $match: {
          category: { $in: categories }
        }
      });
    }

    pipeline.push({
      $addFields: {
        similarityScore: { $meta: "vectorSearchScore" }
      }
    });

    let recommendations = await this.items.aggregate(pipeline).toArray();

    // Appliquer diversité
    if (diversityBoost > 0) {
      recommendations = this.applyDiversityBoost(
        recommendations,
        diversityBoost
      );
    }

    return recommendations.slice(0, limit).map(item => ({
      itemId: item._id,
      name: item.name,
      category: item.category,
      score: item.similarityScore,
      reason: this.generateRecommendationReason(item, user)
    }));
  }

  applyDiversityBoost(recommendations, boost) {
    // Pénaliser items trop similaires entre eux
    const diversified = [];
    const selectedCategories = new Set();

    for (const item of recommendations) {
      let score = item.similarityScore;

      // Pénaliser si catégorie déjà présente
      if (selectedCategories.has(item.category)) {
        score *= (1 - boost);
      } else {
        selectedCategories.add(item.category);
      }

      diversified.push({
        ...item,
        diversifiedScore: score
      });
    }

    return diversified.sort((a, b) =>
      b.diversifiedScore - a.diversifiedScore
    );
  }

  generateRecommendationReason(item, user) {
    // Générer explication de la recommandation
    const reasons = [];

    if (user.topCategories?.includes(item.category)) {
      reasons.push(`Basé sur votre intérêt pour ${item.category}`);
    }

    if (item.rating > 4.5) {
      reasons.push('Très bien noté par d\'autres utilisateurs');
    }

    if (item.trending) {
      reasons.push('Populaire en ce moment');
    }

    return reasons.join('. ');
  }

  async similarItems(itemId, limit = 10) {
    // Trouver items similaires
    const item = await this.items.findOne({ _id: itemId });
    if (!item || !item.embedding) {
      throw new Error('Item or embedding not found');
    }

    return await this.items.aggregate([
      {
        $vectorSearch: {
          index: "item_vector_index",
          path: "embedding",
          queryVector: item.embedding,
          numCandidates: limit * 5,
          limit: limit + 1
        }
      },
      {
        $match: {
          _id: { $ne: itemId }
        }
      },
      { $limit: limit },
      {
        $addFields: {
          similarityScore: { $meta: "vectorSearchScore" }
        }
      },
      {
        $project: {
          name: 1,
          category: 1,
          price: 1,
          image: 1,
          similarityScore: 1
        }
      }
    ]).toArray();
  }

  async hybridRecommendations(userId, options = {}) {
    const { limit = 10 } = options;

    // Combiner plusieurs stratégies
    const [
      collaborative,
      contentBased,
      trending
    ] = await Promise.all([
      this.collaborativeFiltering(userId, limit),
      this.recommendForUser(userId, { limit }),
      this.getTrendingItems(limit)
    ]);

    // Fusionner avec poids
    const scoreMap = new Map();

    const addToMap = (items, weight) => {
      items.forEach(item => {
        const id = item.itemId.toString();
        const existing = scoreMap.get(id);

        if (existing) {
          existing.score += item.score * weight;
          existing.sources.push(weight === 0.4 ? 'collaborative' :
                               weight === 0.4 ? 'content' : 'trending');
        } else {
          scoreMap.set(id, {
            ...item,
            score: item.score * weight,
            sources: [weight === 0.4 ? 'collaborative' :
                     weight === 0.4 ? 'content' : 'trending']
          });
        }
      });
    };

    addToMap(collaborative, 0.4);
    addToMap(contentBased, 0.4);
    addToMap(trending, 0.2);

    return Array.from(scoreMap.values())
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }

  async collaborativeFiltering(userId, limit) {
    // Filtrage collaboratif simplifié
    // Trouver utilisateurs similaires
    const user = await this.users.findOne({ _id: userId });
    if (!user || !user.preferenceEmbedding) {
      return [];
    }

    // Chercher utilisateurs avec profils similaires
    const similarUsers = await this.users.aggregate([
      {
        $vectorSearch: {
          index: "user_vector_index",
          path: "preferenceEmbedding",
          queryVector: user.preferenceEmbedding,
          numCandidates: 50,
          limit: 10
        }
      },
      {
        $match: {
          _id: { $ne: userId }
        }
      }
    ]).toArray();

    // Items aimés par utilisateurs similaires
    const similarUserIds = similarUsers.map(u => u._id);

    const interactions = await this.interactions.aggregate([
      {
        $match: {
          userId: { $in: similarUserIds },
          type: { $in: ['like', 'purchase'] }
        }
      },
      {
        $group: {
          _id: '$itemId',
          count: { $sum: 1 },
          avgRating: { $avg: '$rating' }
        }
      },
      { $sort: { count: -1 } },
      { $limit: limit }
    ]).toArray();

    // Enrichir avec détails items
    const itemIds = interactions.map(i => i._id);
    const items = await this.items.find({
      _id: { $in: itemIds }
    }).toArray();

    return items.map(item => ({
      itemId: item._id,
      name: item.name,
      score: interactions.find(i => i._id.equals(item._id)).count / 10,
      reason: 'Aimé par des utilisateurs avec des goûts similaires'
    }));
  }

  async getTrendingItems(limit) {
    // Items tendance basés sur interactions récentes
    const last7Days = new Date(Date.now() - 7 * 86400000);

    const trending = await this.interactions.aggregate([
      {
        $match: {
          timestamp: { $gte: last7Days },
          type: { $in: ['view', 'like', 'purchase'] }
        }
      },
      {
        $group: {
          _id: '$itemId',
          viewCount: {
            $sum: { $cond: [{ $eq: ['$type', 'view'] }, 1, 0] }
          },
          likeCount: {
            $sum: { $cond: [{ $eq: ['$type', 'like'] }, 1, 0] }
          },
          purchaseCount: {
            $sum: { $cond: [{ $eq: ['$type', 'purchase'] }, 1, 0] }
          }
        }
      },
      {
        $addFields: {
          trendingScore: {
            $add: [
              '$viewCount',
              { $multiply: ['$likeCount', 3] },
              { $multiply: ['$purchaseCount', 5] }
            ]
          }
        }
      },
      { $sort: { trendingScore: -1 } },
      { $limit: limit }
    ]).toArray();

    const itemIds = trending.map(t => t._id);
    const items = await this.items.find({
      _id: { $in: itemIds }
    }).toArray();

    return items.map(item => ({
      itemId: item._id,
      name: item.name,
      score: trending.find(t => t._id.equals(item._id)).trendingScore / 100,
      reason: 'Tendance cette semaine'
    }));
  }

  async getPopularItems(limit) {
    // Fallback: items les plus populaires globalement
    return await this.items.find()
      .sort({ viewCount: -1, rating: -1 })
      .limit(limit)
      .toArray();
  }

  async explainRecommendation(userId, itemId) {
    // Expliquer pourquoi un item est recommandé
    const user = await this.users.findOne({ _id: userId });
    const item = await this.items.findOne({ _id: itemId });

    if (!user || !item) {
      return null;
    }

    // Calculer similarité
    const similarity = this.cosineSimilarity(
      user.preferenceEmbedding,
      item.embedding
    );

    // Analyser historique utilisateur
    const userInteractions = await this.interactions.find({
      userId,
      type: { $in: ['like', 'purchase'] }
    })
      .sort({ timestamp: -1 })
      .limit(10)
      .toArray();

    const interactedItemIds = userInteractions.map(i => i.itemId);
    const interactedItems = await this.items.find({
      _id: { $in: interactedItemIds }
    }).toArray();

    // Trouver catégories communes
    const userCategories = interactedItems.map(i => i.category);
    const categoryMatch = userCategories.includes(item.category);

    return {
      similarity: similarity.toFixed(3),
      categoryMatch,
      reasons: [
        `Similarité: ${(similarity * 100).toFixed(1)}%`,
        categoryMatch ? `Vous aimez ${item.category}` : null,
        item.rating > 4.5 ? 'Très bien noté' : null
      ].filter(Boolean)
    };
  }

  cosineSimilarity(vec1, vec2) {
    const dotProduct = vec1.reduce((sum, val, i) => sum + val * vec2[i], 0);
    const mag1 = Math.sqrt(vec1.reduce((sum, val) => sum + val * val, 0));
    const mag2 = Math.sqrt(vec2.reduce((sum, val) => sum + val * val, 0));
    return dotProduct / (mag1 * mag2);
  }
}

// Utilisation
const recommender = new IntelligentRecommendationEngine(db, process.env.OPENAI_API_KEY);
await recommender.setupVectorIndexes();

// Construire profil utilisateur
await recommender.buildUserProfile(userId);

// Recommandations personnalisées
const recommendations = await recommender.recommendForUser(userId, {
  limit: 10,
  excludeInteracted: true,
  diversityBoost: 0.3
});

console.log('Recommendations:', recommendations);

// Items similaires
const similar = await recommender.similarItems(itemId, 5);

// Recommandations hybrides
const hybrid = await recommender.hybridRecommendations(userId);

// Explication
const explanation = await recommender.explainRecommendation(userId, itemId);
console.log('Why recommended:', explanation);
```

---

## Performance et optimisations

### Dimensionnement et coûts

```javascript
class VectorSearchOptimizer {
  calculateIndexSize(numDocuments, embeddingDimensions) {
    // Estimation taille index vectoriel
    const bytesPerVector = embeddingDimensions * 4;  // float32
    const overhead = 1.3;  // 30% overhead pour index structures

    const totalBytes = numDocuments * bytesPerVector * overhead;
    const totalMB = totalBytes / (1024 * 1024);
    const totalGB = totalMB / 1024;

    return {
      totalBytes,
      totalMB: totalMB.toFixed(2),
      totalGB: totalGB.toFixed(2)
    };
  }

  estimateEmbeddingCosts(numDocuments, provider = 'openai') {
    const costs = {
      openai: {
        'text-embedding-3-small': 0.00002 / 1000,  // per token
        'text-embedding-3-large': 0.00013 / 1000
      },
      cohere: {
        'embed-english-v3.0': 0.0001 / 1000
      }
    };

    const avgTokensPerDoc = 500;
    const totalTokens = numDocuments * avgTokensPerDoc;

    const modelCosts = costs[provider];
    return Object.entries(modelCosts).map(([model, costPerToken]) => ({
      model,
      totalCost: (totalTokens * costPerToken).toFixed(2),
      costPer1M: (1000000 * avgTokensPerDoc * costPerToken).toFixed(2)
    }));
  }

  optimizeBatchSize(embeddingDimensions, availableRAM) {
    // Calculer batch size optimal pour traitement
    const bytesPerEmbedding = embeddingDimensions * 4;
    const ramForEmbeddings = availableRAM * 0.5;  // 50% de RAM disponible

    const optimalBatchSize = Math.floor(
      ramForEmbeddings / bytesPerEmbedding
    );

    return Math.min(optimalBatchSize, 1000);  // Cap à 1000
  }
}

// Utilisation
const optimizer = new VectorSearchOptimizer();

// Estimation taille index
const indexSize = optimizer.calculateIndexSize(1000000, 1536);
console.log('Index size:', indexSize);
// { totalGB: "7.63" }

// Estimation coûts embeddings
const costs = optimizer.estimateEmbeddingCosts(100000, 'openai');
console.log('Embedding costs:', costs);
```

### Stratégies de cache

```javascript
class EmbeddingCache {
  constructor(redisClient) {
    this.redis = redisClient;
    this.ttl = 2592000;  // 30 jours
  }

  generateKey(text, model) {
    const crypto = require('crypto');
    const hash = crypto
      .createHash('sha256')
      .update(`${model}:${text}`)
      .digest('hex');
    return `emb:${hash}`;
  }

  async get(text, model) {
    const key = this.generateKey(text, model);
    const cached = await this.redis.get(key);

    if (cached) {
      return {
        hit: true,
        embedding: JSON.parse(cached)
      };
    }

    return { hit: false };
  }

  async set(text, model, embedding) {
    const key = this.generateKey(text, model);
    await this.redis.setex(
      key,
      this.ttl,
      JSON.stringify(embedding)
    );
  }

  async getOrGenerate(text, model, generator) {
    const cached = await this.get(text, model);

    if (cached.hit) {
      return { embedding: cached.embedding, cached: true };
    }

    const embedding = await generator(text);
    await this.set(text, model, embedding);

    return { embedding, cached: false };
  }
}
```

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Normaliser embeddings si nécessaire
function normalizeEmbedding(embedding) {
  const magnitude = Math.sqrt(
    embedding.reduce((sum, val) => sum + val * val, 0)
  );
  return embedding.map(val => val / magnitude);
}

// 2. Utiliser cache pour embeddings fréquents
const cache = new EmbeddingCache(redisClient);
const { embedding } = await cache.getOrGenerate(
  text,
  model,
  async (t) => await generateEmbedding(t)
);

// 3. Batch processing pour volume
const embeddings = await generateEmbeddingsBatch(texts);

// 4. Filtres pré-recherche pour performance
db.collection.aggregate([
  {
    $vectorSearch: {
      // ...
      filter: {
        category: { $eq: "tech" }
      }
    }
  }
]);

// 5. Monitorer coûts API
let totalTokens = 0;
totalTokens += countTokens(text);
console.log('API cost:', totalTokens * COST_PER_TOKEN);
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas générer embeddings en temps réel pour chaque recherche
// ❌ LENT
for (const doc of documents) {
  doc.embedding = await generateEmbedding(doc.content);
}

// 2. Ne pas stocker embeddings sans index
// ❌ Recherche impossible
await db.collection.insertOne({
  content: text,
  embedding: emb
  // Pas d'index vectoriel configuré
});

// 3. Ne pas mélanger dimensions d'embeddings
// ❌ ERREUR
await db.collection.insertOne({
  embedding: embedding384  // 384 dimensions
});
await db.collection.insertOne({
  embedding: embedding1536  // 1536 dimensions - incompatible!
});

// 4. Ne pas ignorer le pré-filtrage
// ❌ LENT sur gros volumes
$vectorSearch: {
  // Pas de filter - scan tous les documents
}

// 5. Ne pas oublier la gestion d'erreurs API
// ❌ Pas de retry logic
const emb = await generateEmbedding(text);  // Peut échouer
```

---

## Conclusion

Vector Search avec MongoDB Atlas offre :
- ✅ **Recherche sémantique** (sens, pas mots-clés)
- ✅ **Intégration AI native** (OpenAI, Cohere, etc.)
- ✅ **RAG pour LLMs** (context retrieval)
- ✅ **Recommandations intelligentes**
- ✅ **Recherche multimodale**
- ✅ **Scalabilité enterprise**

**Points clés à retenir :**
1. Embeddings capturent le sens sémantique
2. Similarité cosinus pour comparaison
3. Index vectoriel requis (Atlas Search)
4. Cache embeddings pour économie
5. Batch processing pour volume
6. Pré-filtrage pour performance

**Cas d'usage idéaux :**
- RAG pour chatbots/assistants
- Recherche sémantique de documents
- Systèmes de recommandation
- Q&A avec contexte
- Analyse de sentiment
- Clustering de contenu

**Technologies complémentaires :**
- OpenAI/Cohere pour embeddings
- LangChain pour orchestration
- Redis pour cache
- Monitoring des coûts API

---


⏭️ [MongoDB et GraphQL](/16-fonctionnalites-avancees/10-mongodb-graphql.md)
