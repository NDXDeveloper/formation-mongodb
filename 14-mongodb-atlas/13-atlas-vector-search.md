🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.13 Atlas Vector Search

## Introduction

**Atlas Vector Search** transforme MongoDB en **vector database** pour les applications d'IA générative. Stockez des embeddings vectoriels, effectuez des recherches de similarité sémantique, et implémentez des patterns RAG (Retrieval Augmented Generation) directement dans votre base MongoDB. Plus besoin de Pinecone, Weaviate ou Qdrant : vos données et vos vecteurs cohabitent dans la même base, avec la même sécurité et scalabilité.

### 🎯 Objectifs de cette Section

- Comprendre les embeddings et la recherche vectorielle
- Configurer un index vectoriel dans Atlas
- Effectuer des recherches de similarité sémantique
- Implémenter le pattern RAG (Retrieval Augmented Generation)
- Intégrer avec OpenAI, HuggingFace, Cohere
- Optimiser les performances vectorielles
- Construire des applications GenAI production-ready

---

## 🧠 Concepts Fondamentaux

### Qu'est-ce qu'un Embedding ?

```
┌────────────────────────────────────────────────────────────────────────┐
│                    EMBEDDINGS EXPLIQUÉS                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  TEXTE → VECTEUR NUMÉRIQUE (Embedding)                                 │
│                                                                        │
│  Input:  "Le chat dort sur le canapé"                                  │
│  ↓                                                                     │
│  Embedding Model (ex: OpenAI text-embedding-3-small)                   │
│  ↓                                                                     │
│  Output: [0.023, -0.891, 0.445, ..., 0.234]  (1536 dimensions)         │
│           ↑                                                            │
│           Vecteur numérique qui capture le sens sémantique             │
│                                                                        │
│  ───────────────────────────────────────────────────────────────────── │
│                                                                        │
│  PROPRIÉTÉ CLÉ: Similarité Sémantique                                  │
│                                                                        │
│  Textes similaires → Vecteurs proches                                  │
│                                                                        │
│  "Le chat dort"           → [0.02, -0.89, 0.45, ...]                   │
│  "Un félin qui sommeille" → [0.03, -0.88, 0.46, ...]  ← Proche!        │
│  "J'achète une voiture"   → [-0.65, 0.12, -0.33, ...] ← Loin           │
│                                                                        │
│  Distance vectorielle (cosinus, euclidienne) = Similarité sémantique   │
│                                                                        │
│  ───────────────────────────────────────────────────────────────────── │
│                                                                        │
│  DIMENSIONS TYPIQUES:                                                  │
│  • OpenAI text-embedding-3-small: 1536 dimensions                      │
│  • OpenAI text-embedding-3-large: 3072 dimensions                      │
│  • HuggingFace all-MiniLM-L6-v2: 384 dimensions                        │
│  • Cohere embed-english-v3.0: 1024 dimensions                          │
│                                                                        │
│  Plus de dimensions = Plus de nuances capturées (mais plus cher)       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Architecture Vector Search

```
┌────────────────────────────────────────────────────────────────────────┐
│                 ATLAS VECTOR SEARCH ARCHITECTURE                       │
├────────────────────────────────────────────────────────────────────────┤
│
│  APPLICATION
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  1. User Query: "Comment cuisiner des pâtes carbonara?"
│  └────────────────────────────┬─────────────────────────────────────┘
│                               │
│                               ▼
│  EMBEDDING SERVICE (OpenAI, HuggingFace, etc.)
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  2. Convert query to embedding vector
│  │     Query → [0.123, -0.456, 0.789, ..., 0.234]
│  └────────────────────────────┬─────────────────────────────────────┘
│                               │
│                               ▼
│  ATLAS VECTOR SEARCH
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  3. Vector similarity search (k-NN)
│  │     Find k nearest neighbors to query vector
│  │
│  │  ┌──────────────────────────────────────────────────────────────┐
│  │  │ Vector Index (HNSW algorithm)
│  │  │ • Hierarchical graph structure
│  │  │ • Sub-linear search time O(log n)
│  │  │ • Cosine similarity metric
│  │  └──────────────────────────────────────────────────────────────┘
│  └────────────────────────────┬─────────────────────────────────────┘
│                               │
│                               ▼
│  MONGODB ATLAS CLUSTER
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  4. Results: Documents with highest similarity
│  │
│  │  Document 1: "Recette carbonara authentique" (score: 0.92)
│  │  Document 2: "Pâtes à la carbonara rapide" (score: 0.89)
│  │  Document 3: "Carbonara : astuces de chef" (score: 0.85)
│  │
│  │  Each document contains:
│  │  • Content (text)
│  │  • Embedding vector (1536 dims)
│  │  • Metadata (category, date, etc.)
│  └──────────────────────────────────────────────────────────────────┘
│
│  KEY ADVANTAGES:
│  ✅ Unified storage (vectors + data + metadata)
│  ✅ Single database (no separate vector DB)
│  ✅ Hybrid search (vector + metadata filters)
│  ✅ ACID transactions
│  ✅ Same security, scaling, backups as MongoDB
│
└──────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Configuration Vector Index

### Structure de Document

```javascript
// Collection: articles
// Chaque document contient texte + embedding

{
  "_id": ObjectId("..."),

  // Contenu original
  "title": "Guide complet des pâtes italiennes",
  "content": "Les pâtes sont un pilier de la cuisine italienne...",
  "category": "cuisine",
  "author": "Chef Mario",
  "publishedAt": ISODate("2024-11-15"),

  // Embedding vectoriel (généré par OpenAI, HuggingFace, etc.)
  "embedding": [
    0.0234, -0.8912, 0.4456, 0.7823, -0.2341, 0.5678,
    // ... 1536 dimensions au total
    0.8901, 0.2345, -0.6789, 0.1234
  ],

  // Métadonnées pour filtrage hybride
  "tags": ["italien", "recette", "pâtes"],
  "difficulty": "facile",
  "preparationTime": 30
}
```

### Création d'Index Vectoriel

```javascript
// Configuration Vector Search Index via Atlas UI ou API

{
  "name": "vector_index",
  "type": "vectorSearch",

  "fields": [
    {
      "type": "vector",
      "path": "embedding",           // Champ contenant le vecteur
      "numDimensions": 1536,          // Dimensions (OpenAI small = 1536)
      "similarity": "cosine"          // Métrique: cosine, euclidean, dotProduct
    },

    // Champs supplémentaires pour filtrage hybride (optionnel)
    {
      "type": "filter",
      "path": "category"
    },
    {
      "type": "filter",
      "path": "publishedAt"
    }
  ]
}
```

### Configuration via MongoDB Shell

```javascript
// Créer l'index vectoriel

db.articles.createSearchIndex(
  "vector_index",
  "vectorSearch",
  {
    fields: [
      {
        type: "vector",
        path: "embedding",
        numDimensions: 1536,
        similarity: "cosine"
      }
    ]
  }
);

// Vérifier la création
db.articles.listSearchIndexes();
```

---

## 🔍 Requêtes Vector Search

### Recherche Basique de Similarité

```javascript
// ÉTAPE 1: Générer embedding pour la requête
const queryText = "Comment faire des pâtes carbonara?";

// Appel API OpenAI (ou autre service)
const embeddingResponse = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: queryText
});

const queryVector = embeddingResponse.data[0].embedding;
// queryVector = [0.123, -0.456, ..., 0.789] (1536 dimensions)

// ÉTAPE 2: Recherche vectorielle dans MongoDB
const results = await db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",           // Nom de l'index
      path: "embedding",                // Champ vectoriel
      queryVector: queryVector,         // Vecteur de la requête
      numCandidates: 100,               // Candidates à examiner
      limit: 10                         // Top 10 résultats
    }
  },
  {
    $project: {
      title: 1,
      content: 1,
      category: 1,
      score: { $meta: "vectorSearchScore" }  // Score de similarité
    }
  }
]).toArray();

// Résultats triés par similarité
console.log(results);
/*
[
  {
    _id: ObjectId("..."),
    title: "Carbonara authentique: recette italienne",
    content: "La vraie carbonara se fait avec...",
    category: "cuisine",
    score: 0.923  // Très similaire!
  },
  {
    _id: ObjectId("..."),
    title: "Pâtes italiennes: le guide complet",
    content: "Les pâtes sont un pilier...",
    category: "cuisine",
    score: 0.887
  },
  ...
]
*/
```

### Recherche Hybride (Vector + Filtres)

```javascript
// Combinaison: Similarité vectorielle + Filtres métadonnées

const results = await db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryVector,
      numCandidates: 200,
      limit: 10,

      // Filtres additionnels (pre-filter)
      filter: {
        category: "cuisine",
        difficulty: "facile",
        preparationTime: { $lte: 30 }
      }
    }
  },
  {
    $project: {
      title: 1,
      content: 1,
      difficulty: 1,
      preparationTime: 1,
      score: { $meta: "vectorSearchScore" }
    }
  }
]).toArray();

// Résultats: Recettes de pâtes SIMILAIRES + Faciles + Rapides
```

### Fonction Helper Réutilisable

```javascript
// Helper function pour recherche vectorielle

async function vectorSearch(
  collection,
  queryText,
  options = {}
) {
  const {
    indexName = "vector_index",
    limit = 10,
    numCandidates = 100,
    filter = {},
    embeddingModel = "text-embedding-3-small"
  } = options;

  // 1. Générer embedding
  const embeddingResponse = await openai.embeddings.create({
    model: embeddingModel,
    input: queryText
  });

  const queryVector = embeddingResponse.data[0].embedding;

  // 2. Recherche vectorielle
  const pipeline = [
    {
      $vectorSearch: {
        index: indexName,
        path: "embedding",
        queryVector: queryVector,
        numCandidates: numCandidates,
        limit: limit,
        ...(Object.keys(filter).length > 0 && { filter })
      }
    },
    {
      $project: {
        _id: 1,
        title: 1,
        content: 1,
        metadata: 1,
        score: { $meta: "vectorSearchScore" }
      }
    }
  ];

  return await collection.aggregate(pipeline).toArray();
}

// Usage
const results = await vectorSearch(
  db.collection("articles"),
  "recettes italiennes rapides",
  {
    limit: 5,
    filter: { category: "cuisine", difficulty: "facile" }
  }
);
```

---

## 🤖 Pattern RAG (Retrieval Augmented Generation)

### Architecture RAG

```
┌────────────────────────────────────────────────────────────────────────┐
│                       RAG PATTERN ARCHITECTURE                         │
├────────────────────────────────────────────────────────────────────────┤
│
│  RAG = Retrieval Augmented Generation
│  Améliore les LLMs avec votre propre base de connaissances
│
│  WORKFLOW:
│  ───────────
│
│  1. USER QUERY
│  ┌──────────────────────────────────────────────────────────────────┐
│  │ "Quel est le processus de remboursement de votre entreprise?"    │
│  └────────────────────────────┬──────────────────────────────────────
│                               │
│                               ▼
│  2. VECTOR SEARCH (Retrieval)
│  ┌──────────────────────────────────────────────────────────────────┐
│  │ Query → Embedding → Vector Search
│  │
│  │ Top 3 documents pertinents:
│  │ • "Politique de remboursement" (score: 0.94)
│  │ • "Guide remboursement employés" (score: 0.89)
│  │ • "FAQ finance" (score: 0.82)
│  └────────────────────────────┬───────────────────────────────────────┘
│                               │
│                               ▼
│  3. CONTEXT ASSEMBLY
│  ┌──────────────────────────────────────────────────────────────────┐
│  │ Construct prompt avec contexte:
│  │
│  │ System: "Tu es un assistant qui répond en te basant
│  │          uniquement sur le contexte fourni."
│  │
│  │ Context: [Documents retrieved]
│  │
│  │ Question: [User query]
│  └────────────────────────────┬───────────────────────────────────────┘
│                               │
│                               ▼
│  4. LLM GENERATION
│  ┌──────────────────────────────────────────────────────────────────┐
│  │ OpenAI GPT-4 / Claude / etc.
│  │
│  │ Génère réponse basée sur le contexte récupéré
│  └────────────────────────────┬───────────────────────────────────────┘
│                               │
│                               ▼
│  5. RESPONSE
│  ┌──────────────────────────────────────────────────────────────────┐
│  │ "Pour obtenir un remboursement, suivez ces étapes:
│  │  1. Soumettez une demande via le portail employé
│  │  2. Joignez les reçus justificatifs
│  │  3. Validation sous 48h par votre manager
│  │  4. Paiement sous 5 jours ouvrés
│  │
│  │  Source: Politique de remboursement (page 12)"
│  └──────────────────────────────────────────────────────────────────┘
│
│  AVANTAGES RAG:
│  ✅ Réponses basées sur VOS données (pas hallucinations)
│  ✅ Toujours à jour (nouveaux docs = nouvelles réponses)
│  ✅ Sources traçables (citations)
│  ✅ Pas de fine-tuning LLM requis
│  ✅ Cost-effective
│
└────────────────────────────────────────────────────────────────────────┘
```

### Implémentation RAG Complète

```javascript
// RAG Implementation avec Atlas Vector Search + OpenAI

import { MongoClient } from "mongodb";
import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const client = new MongoClient(process.env.MONGODB_URI);

async function ragQuery(userQuestion) {
  try {
    await client.connect();
    const db = client.db("knowledge_base");
    const collection = db.collection("documents");

    // ÉTAPE 1: Generate embedding pour la question
    console.log("Generating query embedding...");
    const embeddingResponse = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: userQuestion
    });
    const queryVector = embeddingResponse.data[0].embedding;

    // ÉTAPE 2: Vector search pour documents pertinents
    console.log("Searching for relevant documents...");
    const relevantDocs = await collection.aggregate([
      {
        $vectorSearch: {
          index: "vector_index",
          path: "embedding",
          queryVector: queryVector,
          numCandidates: 100,
          limit: 3  // Top 3 documents les plus pertinents
        }
      },
      {
        $project: {
          title: 1,
          content: 1,
          metadata: 1,
          score: { $meta: "vectorSearchScore" }
        }
      }
    ]).toArray();

    console.log(`Found ${relevantDocs.length} relevant documents`);

    // ÉTAPE 3: Construire contexte
    const context = relevantDocs
      .map((doc, i) => `Document ${i + 1}: ${doc.title}\n${doc.content}`)
      .join("\n\n---\n\n");

    // ÉTAPE 4: Construire prompt pour LLM
    const systemPrompt = `Tu es un assistant intelligent qui répond aux questions en te basant UNIQUEMENT sur le contexte fourni.
Si la réponse n'est pas dans le contexte, dis "Je ne trouve pas cette information dans les documents disponibles."
Cite toujours tes sources en mentionnant le numéro du document.`;

    const userPrompt = `Contexte:
${context}

Question: ${userQuestion}

Réponds de manière claire et concise, en citant tes sources.`;

    // ÉTAPE 5: Appel LLM
    console.log("Generating answer with LLM...");
    const completion = await openai.chat.completions.create({
      model: "gpt-4o",
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userPrompt }
      ],
      temperature: 0.3  // Bas = plus factuel
    });

    const answer = completion.choices[0].message.content;

    // ÉTAPE 6: Retourner réponse + sources
    return {
      answer: answer,
      sources: relevantDocs.map(doc => ({
        title: doc.title,
        score: doc.score,
        metadata: doc.metadata
      }))
    };

  } finally {
    await client.close();
  }
}

// Usage
const result = await ragQuery(
  "Quelle est la politique de remboursement de l'entreprise?"
);

console.log("Answer:", result.answer);
console.log("\nSources:");
result.sources.forEach((source, i) => {
  console.log(`${i + 1}. ${source.title} (relevance: ${source.score.toFixed(2)})`);
});

/* Output exemple:
Answer: Pour obtenir un remboursement, vous devez suivre ces étapes selon le Document 1 (Politique de remboursement):
1. Soumettez une demande via le portail employé avec justificatifs
2. Validation par votre manager sous 48h
3. Paiement sous 5 jours ouvrés maximum

Les plafonds sont détaillés dans le Document 2, avec 50€/jour pour les repas et 100€/nuit pour l'hébergement.

Sources:
1. Politique de remboursement employés (relevance: 0.94)
2. Guide frais professionnels (relevance: 0.89)
3. FAQ finance (relevance: 0.82)
*/
```

---

## 🎯 Cas d'Usage Avancés

### 1. Semantic Search (Moteur de Recherche)

```javascript
// Use case: Moteur de recherche intelligent pour documentation

async function semanticSearch(query, filters = {}) {
  // Contrairement à la recherche textuelle classique,
  // semantic search comprend l'INTENTION

  // Query: "réparer bug serveur crashé"
  // Trouve aussi: "résoudre problème démarrage", "serveur ne répond pas"
  // Car sémantiquement similaire!

  const embedding = await generateEmbedding(query);

  return await db.documentation.aggregate([
    {
      $vectorSearch: {
        index: "semantic_index",
        path: "content_embedding",
        queryVector: embedding,
        numCandidates: 200,
        limit: 20,
        filter: filters
      }
    },
    {
      $project: {
        title: 1,
        summary: 1,
        url: 1,
        category: 1,
        score: { $meta: "vectorSearchScore" },
        // Highlight du contenu pertinent
        snippet: { $substr: ["$content", 0, 200] }
      }
    }
  ]).toArray();
}
```

### 2. Recommendation System

```javascript
// Use case: Recommandations de produits basées sur similarité

async function recommendSimilarProducts(productId, limit = 5) {
  // Trouver le produit source
  const product = await db.products.findOne({ _id: productId });

  if (!product || !product.embedding) {
    throw new Error("Product not found or missing embedding");
  }

  // Chercher produits similaires
  const similar = await db.products.aggregate([
    {
      $vectorSearch: {
        index: "product_vector_index",
        path: "embedding",
        queryVector: product.embedding,
        numCandidates: 100,
        limit: limit + 1  // +1 car le produit lui-même sera dans les résultats
      }
    },
    {
      $match: {
        _id: { $ne: productId }  // Exclure le produit source
      }
    },
    {
      $project: {
        name: 1,
        description: 1,
        price: 1,
        category: 1,
        image: 1,
        similarity: { $meta: "vectorSearchScore" }
      }
    },
    {
      $limit: limit
    }
  ]).toArray();

  return similar;
}

// Frontend usage
const recommendations = await recommendSimilarProducts("prod_12345", 4);
// Affiche "Produits similaires" sur la page produit
```

### 3. Chatbot avec Mémoire Conversationnelle

```javascript
// Use case: Chatbot intelligent avec contexte

class RAGChatbot {
  constructor(db, knowledgeCollection) {
    this.db = db;
    this.collection = knowledgeCollection;
    this.conversationHistory = [];
  }

  async chat(userMessage) {
    // Ajouter message utilisateur à l'historique
    this.conversationHistory.push({
      role: "user",
      content: userMessage
    });

    // Chercher contexte pertinent
    const embedding = await generateEmbedding(userMessage);
    const relevantDocs = await this.collection.aggregate([
      {
        $vectorSearch: {
          index: "knowledge_index",
          path: "embedding",
          queryVector: embedding,
          numCandidates: 50,
          limit: 2
        }
      }
    ]).toArray();

    // Construire prompt avec historique + contexte
    const context = relevantDocs
      .map(doc => doc.content)
      .join("\n\n");

    const messages = [
      {
        role: "system",
        content: `Tu es un assistant qui aide les utilisateurs.
Contexte disponible:
${context}

Réponds en te basant sur ce contexte et l'historique de conversation.`
      },
      ...this.conversationHistory
    ];

    // Appel LLM
    const completion = await openai.chat.completions.create({
      model: "gpt-4o",
      messages: messages
    });

    const assistantMessage = completion.choices[0].message.content;

    // Ajouter réponse à l'historique
    this.conversationHistory.push({
      role: "assistant",
      content: assistantMessage
    });

    return {
      message: assistantMessage,
      sources: relevantDocs.map(d => d.title)
    };
  }

  clearHistory() {
    this.conversationHistory = [];
  }
}

// Usage
const chatbot = new RAGChatbot(db, db.collection("knowledge"));

await chatbot.chat("Comment fonctionne votre API?");
// → Réponse avec contexte

await chatbot.chat("Et quel est le rate limit?");
// → Comprend que "rate limit" se réfère à l'API
//   grâce à l'historique conversationnel
```

### 4. Duplicate Detection

```javascript
// Use case: Détection de doublons sémantiques

async function findDuplicates(newDocument, threshold = 0.95) {
  // Générer embedding du nouveau document
  const embedding = await generateEmbedding(newDocument.content);

  // Chercher documents très similaires
  const potentialDuplicates = await db.articles.aggregate([
    {
      $vectorSearch: {
        index: "content_vector_index",
        path: "embedding",
        queryVector: embedding,
        numCandidates: 100,
        limit: 10
      }
    },
    {
      $match: {
        score: { $gte: threshold }  // Similarité > 95%
      }
    },
    {
      $project: {
        title: 1,
        content: 1,
        publishedAt: 1,
        similarity: { $meta: "vectorSearchScore" }
      }
    }
  ]).toArray();

  if (potentialDuplicates.length > 0) {
    console.log(`⚠️ Warning: Found ${potentialDuplicates.length} potential duplicates`);
    return potentialDuplicates;
  }

  return [];
}

// Avant d'insérer un nouvel article
const duplicates = await findDuplicates(newArticle);
if (duplicates.length > 0) {
  // Alerter l'éditeur ou fusionner automatiquement
}
```

---

## 🚀 Performance et Optimisation

### Choix du Nombre de Dimensions

```
┌────────────────────────────────────────────────────────────────────────┐
│                  EMBEDDING DIMENSIONS TRADE-OFFS
├───────────────────────────────────────────────────────────────────────
│
│  MODEL                         DIMS    QUALITY   SPEED    COST
│  ─────────────────────────────────────────────────────────────────────
│  text-embedding-3-small        1536   ⭐⭐⭐     ⭐⭐⭐⭐   $
│  text-embedding-3-large        3072   ⭐⭐⭐⭐⭐   ⭐⭐⭐     $$
│  all-MiniLM-L6-v2 (HF)         384    ⭐⭐       ⭐⭐⭐⭐⭐ Free
│  Cohere embed-english-v3       1024   ⭐⭐⭐⭐    ⭐⭐⭐     $
│
│  RECOMMANDATIONS:
│  • Prototype/MVP: all-MiniLM-L6-v2 (gratuit, rapide)
│  • Production générale: text-embedding-3-small (bon équilibre)
│  • Haute précision: text-embedding-3-large (meilleure qualité)
│
│  IMPACT STORAGE:
│  • 1 million docs × 1536 dims × 4 bytes = ~6 GB
│  • 1 million docs × 3072 dims × 4 bytes = ~12 GB
│  • 1 million docs × 384 dims × 4 bytes = ~1.5 GB
│
└───────────────────────────────────────────────────────────────────────┘
```

### Optimisation de numCandidates

```javascript
// numCandidates = nombre de candidats examinés avant sélection finale

// ❌ MAUVAIS: Trop bas (qualité médiocre)
{
  $vectorSearch: {
    queryVector: embedding,
    numCandidates: 20,  // ❌ Trop peu
    limit: 10
  }
}
// Problème: Peut manquer de bons résultats

// ✅ BON: Équilibre qualité/performance
{
  $vectorSearch: {
    queryVector: embedding,
    numCandidates: 100,  // ✅ Good: 10x limit
    limit: 10
  }
}

// ⚠️ ACCEPTABLE: Plus de qualité, moins de perf
{
  $vectorSearch: {
    queryVector: embedding,
    numCandidates: 500,  // ⚠️ 50x limit = plus lent
    limit: 10
  }
}

// RÈGLE GÉNÉRALE:
// numCandidates = limit × 10 (bon équilibre)
// numCandidates = limit × 50 (haute précision)
```

### Stratégies de Chunking

```javascript
// Pour longs documents: découper en chunks

function chunkDocument(document, chunkSize = 500, overlap = 50) {
  const words = document.content.split(' ');
  const chunks = [];

  for (let i = 0; i < words.length; i += chunkSize - overlap) {
    const chunk = words.slice(i, i + chunkSize).join(' ');

    chunks.push({
      documentId: document._id,
      chunkIndex: chunks.length,
      content: chunk,
      metadata: {
        title: document.title,
        author: document.author,
        // ... métadonnées héritées
      }
    });
  }

  return chunks;
}

// Générer embeddings pour chaque chunk
async function embedChunks(chunks) {
  const embeddings = await Promise.all(
    chunks.map(chunk => generateEmbedding(chunk.content))
  );

  return chunks.map((chunk, i) => ({
    ...chunk,
    embedding: embeddings[i]
  }));
}

// Insérer chunks avec embeddings
const chunks = chunkDocument(longDocument);
const chunksWithEmbeddings = await embedChunks(chunks);
await db.document_chunks.insertMany(chunksWithEmbeddings);

// AVANTAGES:
// ✅ Meilleure granularité
// ✅ Recherche plus précise
// ✅ Moins de tokens LLM (context plus ciblé)
```

---

## 📋 Best Practices

### Checklist Production

```
┌───────────────────────────────────────────────────────────────────────┐
│              VECTOR SEARCH PRODUCTION CHECKLIST                       │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  INDEX CONFIGURATION                                                  │
│  ☐ Choisir nombre de dimensions approprié (balance qualité/coût)      │
│  ☐ Métrique de similarité: cosine (général), euclidean (numérique)    │
│  ☐ Créer index sur champs métadonnées pour filtrage hybride           │
│  ☐ Tester index avec données production-scale                         │
│                                                                       │
│  EMBEDDINGS GENERATION                                                │
│  ☐ Utiliser même modèle pour indexation et recherche                  │
│  ☐ Normaliser texte avant embedding (lowercase, trim)                 │
│  ☐ Implémenter rate limiting pour API embeddings                      │
│  ☐ Cacher embeddings (éviter regénération)                            │
│  ☐ Batch processing pour génération bulk                              │
│                                                                       │
│  QUERIES                                                              │
│  ☐ numCandidates ≥ limit × 10 (balance qualité/perf)                  │
│  ☐ Utiliser filtres métadonnées pour réduire espace recherche         │
│  ☐ Implémenter retry logic (API failures)                             │
│  ☐ Monitorer latence queries (target < 200ms)                         │
│                                                                       │
│  RAG IMPLEMENTATION                                                   │
│  ☐ Valider contexte avant envoi au LLM (pertinence)                   │
│  ☐ Limiter longueur contexte (éviter dépassement token limit)         │
│  ☐ Citer sources dans réponses (traçabilité)                          │
│  ☐ Implémenter fallback si aucun document pertinent                   │
│  ☐ Logger queries pour amélioration continue                          │
│                                                                       │
│  SECURITY                                                             │
│  ☐ Ne pas exposer embeddings côté client                              │
│  ☐ Filtrer résultats selon permissions utilisateur                    │
│  ☐ Valider/sanitizer inputs                                           │
│  ☐ Rate limiting sur endpoints                                        │
│                                                                       │
│  MONITORING                                                           │
│  ☐ Tracker coûts API embeddings                                       │
│  ☐ Monitorer latence vector search                                    │
│  ☐ Analyser qualité résultats (user feedback)                         │
│  ☐ Alertes sur erreurs embeddings/search                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé

### Points Clés

1. **Embeddings**
   - Vecteurs numériques capturant sens sémantique
   - 384-3072 dimensions selon modèle
   - Similarité vectorielle = similarité sémantique

2. **Vector Search**
   - Recherche k-NN (k plus proches voisins)
   - Algorithme HNSW (sub-linear performance)
   - Métriques: cosine, euclidean, dotProduct

3. **RAG Pattern**
   - Retrieval (vector search) + Generation (LLM)
   - Réponses basées sur VOS données
   - Pas d'hallucinations
   - Sources traçables

4. **Use Cases**
   - Semantic search (moteurs recherche)
   - Recommendations (produits similaires)
   - Chatbots intelligents
   - Duplicate detection

5. **Performance**
   - numCandidates = limit × 10 (équilibre)
   - Chunking pour longs documents
   - Filtrage hybride (vector + metadata)
   - Caching embeddings

### Configuration Minimale

```javascript
// 1. Structure document
{
  content: "Votre texte...",
  embedding: [0.023, -0.891, ...],  // 1536 dims
  metadata: { category: "X" }
}

// 2. Index vectoriel
db.collection.createSearchIndex("vector_index", "vectorSearch", {
  fields: [{
    type: "vector",
    path: "embedding",
    numDimensions: 1536,
    similarity: "cosine"
  }]
});

// 3. Recherche
const embedding = await generateEmbedding(query);
db.collection.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: embedding,
      numCandidates: 100,
      limit: 10
    }
  }
]);
```

### Ressources

- [Atlas Vector Search Docs](https://www.mongodb.com/docs/atlas/atlas-vector-search/)
- [OpenAI Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [RAG Best Practices](https://www.mongodb.com/developer/products/atlas/rag-best-practices/)

---


⏭️ [Triggers et fonctions serverless](/14-mongodb-atlas/14-triggers-fonctions-serverless.md)
