🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.4 Éviter les Documents Trop Volumineux

## Introduction

MongoDB impose une limite stricte de 16 MB par document, mais cette limite technique n'est que le sommet de l'iceberg. Bien avant d'atteindre cette limite fatale, des documents volumineux créent des problèmes de performance, de mémoire, de réseau et de maintenabilité qui peuvent paralyser une application. Un document de "seulement" 1-2 MB peut déjà causer des problèmes significatifs en production.

Cette section explore pourquoi la taille des documents est critique, comment identifier et prévenir la croissance excessive, et quels patterns architecturaux utiliser pour gérer efficacement les grandes quantités de données tout en maintenant des documents compacts et performants.

---

## Comprendre les Limites et les Impacts

### La Limite Technique : 16 MB

MongoDB utilise le format BSON (Binary JSON) qui impose une limite stricte :

```javascript
// ❌ Erreur fatale à l'insertion
BSONObj size: 16777217 (0x1000001) is invalid.
Size must be between 0 and 16793600(16MB)
```

**Pourquoi cette limite existe** :
- Prévention des allocations mémoire massives
- Protection contre les attaques par déni de service
- Garantie de performance raisonnable
- Limite de stockage contiguë

### L'Impact Réel Commence Bien Avant

Les problèmes de performance apparaissent progressivement :

| Taille Document | Impact | Symptômes |
|-----------------|--------|-----------|
| **< 100 KB** | ✅ Optimal | Performance nominale |
| **100 KB - 500 KB** | ⚠️ Acceptable | Léger impact sur transfert réseau |
| **500 KB - 1 MB** | ⚠️ Problématique | Latence accrue, cache moins efficace |
| **1 MB - 5 MB** | ❌ Sérieux | Performance dégradée, fragmentation |
| **5 MB - 16 MB** | ❌ Critique | Problèmes majeurs, révision nécessaire |
| **> 16 MB** | 💥 Impossible | Erreur, document rejeté |

---

## ✅ DO : Maintenir les Documents Sous 1 MB

**Explication** : Un seuil de 1 MB devrait être considéré comme la limite pratique pour la majorité des cas d'usage, et idéalement viser moins de 100 KB.

**Seuils recommandés** :
```javascript
// ✅ Tailles cibles
const DOCUMENT_SIZE_LIMITS = {
  OPTIMAL: 100 * 1024,        // 100 KB - Cible idéale
  ACCEPTABLE: 500 * 1024,     // 500 KB - Limite confortable
  WARNING: 1 * 1024 * 1024,   // 1 MB - Seuil d'alerte
  CRITICAL: 5 * 1024 * 1024,  // 5 MB - Révision urgente
  MAXIMUM: 16 * 1024 * 1024   // 16 MB - Limite technique
};

// Fonction de vérification
function validateDocumentSize(doc) {
  const size = JSON.stringify(doc).length;

  if (size > DOCUMENT_SIZE_LIMITS.CRITICAL) {
    throw new Error(`Document too large: ${(size/1024/1024).toFixed(2)} MB`);
  }

  if (size > DOCUMENT_SIZE_LIMITS.WARNING) {
    console.warn(`Document approaching size limit: ${(size/1024).toFixed(2)} KB`);
  }

  return size;
}
```

**Bénéfices de documents compacts** :

### 1. Performance Réseau
```javascript
// Document 10 KB vs 2 MB
// Temps de transfert sur connexion 100 Mbps

10 KB : ~1 ms
2 MB  : ~160 ms
= 160x plus lent
```

### 2. Utilisation Mémoire
```javascript
// MongoDB charge les documents entiers en mémoire
// 1000 requêtes simultanées

Documents 10 KB : 10 MB RAM
Documents 2 MB  : 2 GB RAM
= 200x plus de mémoire
```

### 3. Efficacité du Cache
- Plus de documents tiennent en cache WiredTiger
- Taux de cache hit amélioré
- Moins d'I/O disque

### 4. Rapidité des Opérations
```javascript
// Temps de lecture complète du document
Document 10 KB : <1 ms
Document 2 MB  : 50-100 ms
```

---

## ❌ DON'T : Laisser les Tableaux Croître Indéfiniment

**Explication** : Le pattern le plus dangereux et commun : des tableaux qui s'agrandissent sans limite au fil du temps.

**Anti-pattern critique** :
```javascript
// ❌ DANGER : Tableau qui grossit sans limite
{
  _id: ObjectId("..."),
  userId: "user123",
  username: "alice",
  loginHistory: [
    { date: ISODate("2024-01-01T10:00:00Z"), ip: "192.168.1.1" },
    { date: ISODate("2024-01-02T09:30:00Z"), ip: "192.168.1.1" },
    // ... des milliers d'entrées s'accumulent
    // Document grossit de 100 bytes à chaque login
    // Après 10,000 logins = 1 MB
    // Après 100,000 logins = 10 MB
    // Après 160,000 logins = 16 MB = ERREUR!
  ]
}
```

**Scénarios courants** :
```javascript
// ❌ Historique d'événements illimité
{
  orderId: "ORD123",
  events: [
    { timestamp: "...", action: "created" },
    { timestamp: "...", action: "updated" },
    // Des centaines/milliers d'événements...
  ]
}

// ❌ Commentaires illimités
{
  articleId: "ART456",
  comments: [
    { user: "...", text: "...", date: "..." },
    // Des milliers de commentaires...
  ]
}

// ❌ Messages de conversation illimités
{
  conversationId: "CONV789",
  messages: [
    { from: "...", to: "...", text: "...", date: "..." },
    // Des dizaines de milliers de messages...
  ]
}

// ❌ Logs applicatifs dans le document
{
  processId: "PROC001",
  logs: [
    { level: "info", message: "...", timestamp: "..." },
    // Des millions d'entrées de log...
  ]
}
```

**Conséquences** :

### 1. Croissance Exponentielle
```javascript
// Progression typique d'un document avec array illimité
Jour 1    : 5 KB
Mois 1    : 50 KB
Mois 6    : 500 KB
An 1      : 2 MB
An 2      : 4 MB
An 3      : 8 MB
An 4      : 16 MB = CRASH!
```

### 2. Performance Dégradée
- Lecture complète obligatoire même pour un seul élément
- Temps de parsing JSON explosif
- Saturation mémoire client

### 3. Fragmentation du Stockage
```javascript
// Document initialement : 5 KB, alloué dans un espace de 5 KB
// Après 6 mois : 500 KB, ne tient plus dans l'espace original
// MongoDB doit :
// 1. Allouer 500 KB ailleurs
// 2. Copier le document
// 3. Laisser l'ancien espace vide (fragmentation)
// 4. Mettre à jour tous les index

// Impact :
// - Performance dégradée lors des updates
// - Espace disque gaspillé
// - Besoin de compaction régulière
```

### 4. Impossibilité d'Indexer Efficacement
```javascript
// ❌ Impossible d'indexer efficacement dans un grand tableau
db.articles.createIndex({ "comments.userId": 1 });
// Si 10,000 commentaires par article :
// - Index devient énorme
// - Performance catastrophique
// - Multikey index inefficace
```

---

## ✅ DO : Utiliser le Pattern Bucket pour les Séries Temporelles

**Explication** : Le pattern Bucket limite le nombre d'éléments par document et crée de nouveaux documents au besoin.

**Pattern recommandé** :
```javascript
// ✅ Pattern Bucket : Limiter à 100 événements par document
{
  _id: ObjectId("..."),
  userId: "user123",
  bucketDate: ISODate("2024-01-01T00:00:00Z"), // Début du bucket
  eventCount: 100,  // Compteur
  events: [
    { date: ISODate("2024-01-01T10:00:00Z"), action: "login" },
    { date: ISODate("2024-01-01T10:15:00Z"), action: "view_page" },
    // Maximum 100 événements
  ]
}

// Quand le bucket est plein, créer un nouveau document
{
  _id: ObjectId("..."),
  userId: "user123",
  bucketDate: ISODate("2024-01-02T00:00:00Z"),
  eventCount: 45,
  events: [
    // Nouveaux événements
  ]
}
```

**Implémentation** :
```javascript
// ✅ Fonction d'insertion avec bucketing
async function addUserEvent(userId, event) {
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  const result = await db.userEvents.updateOne(
    {
      userId: userId,
      bucketDate: today,
      eventCount: { $lt: 100 }  // Bucket pas encore plein
    },
    {
      $push: { events: event },
      $inc: { eventCount: 1 }
    },
    { upsert: true }
  );

  // Si aucun document modifié, tous les buckets sont pleins
  if (result.matchedCount === 0) {
    // Créer un nouveau bucket avec un identifiant unique
    await db.userEvents.insertOne({
      userId: userId,
      bucketDate: today,
      bucketSequence: await getNextBucketSequence(userId, today),
      eventCount: 1,
      events: [event]
    });
  }
}
```

**Avantages mesurables** :
```javascript
// Sans bucketing (1 document par user)
// 100,000 événements par user = 10 MB par document
// 1,000 users actifs = 10 GB en mémoire pour requêtes courantes

// Avec bucketing (100 événements max)
// 100 événements par document = ~10 KB par document
// 1,000 buckets (100 événements × 1,000 users) = 10 MB
= 1000x moins de mémoire pour données récentes
```

**Variantes du pattern** :
```javascript
// ✅ Bucket par heure (données haute fréquence)
{
  sensorId: "SENSOR123",
  hourBucket: ISODate("2024-01-15T14:00:00Z"),
  readings: [/* max 3600 lectures */]
}

// ✅ Bucket par taille maximale (plutôt que count)
{
  userId: "user123",
  bucketId: 42,
  maxSizeBytes: 100000,  // 100 KB max
  currentSize: 87234,
  items: [/* données variables */]
}

// ✅ Bucket avec métadonnées agrégées
{
  storeId: "STORE001",
  date: ISODate("2024-01-15"),
  transactionCount: 150,
  totalAmount: 15750.00,
  transactions: [/* max 200 transactions */],
  // Métadonnées pour requêtes rapides sans parser le tableau
  summary: {
    minAmount: 5.00,
    maxAmount: 250.00,
    avgAmount: 105.00
  }
}
```

---

## ✅ DO : Utiliser le Pattern Subset pour Données Partielles

**Explication** : Imbriquer seulement un sous-ensemble des données les plus pertinentes, avec référence vers les données complètes.

**Pattern recommandé** :
```javascript
// ✅ Document article avec subset de commentaires récents
{
  _id: ObjectId("..."),
  title: "Understanding MongoDB Patterns",
  content: "...",
  author: "Alice",

  // Subset : seulement les 10 commentaires les plus récents
  recentComments: [
    {
      _id: ObjectId("..."),
      userId: "user123",
      text: "Great article!",
      createdAt: ISODate("2024-01-15T10:00:00Z")
    },
    // ... 9 autres commentaires récents
  ],

  // Métadonnées
  totalComments: 5247,  // Nombre total
  commentsCollectionRef: "article_comments"  // Référence à collection complète
}

// Collection séparée pour tous les commentaires
// Collection: article_comments
{
  _id: ObjectId("..."),
  articleId: ObjectId("..."),  // Référence à l'article
  userId: "user123",
  text: "Great article!",
  createdAt: ISODate("2024-01-15T10:00:00Z")
}
```

**Implémentation** :
```javascript
// ✅ Maintenir le subset à jour
async function addComment(articleId, comment) {
  // 1. Ajouter à la collection complète
  const commentDoc = await db.article_comments.insertOne({
    articleId: articleId,
    ...comment,
    createdAt: new Date()
  });

  // 2. Ajouter au subset (limité à 10 plus récents)
  await db.articles.updateOne(
    { _id: articleId },
    {
      $push: {
        recentComments: {
          $each: [{
            _id: commentDoc.insertedId,
            ...comment,
            createdAt: new Date()
          }],
          $sort: { createdAt: -1 },
          $slice: 10  // Garder seulement les 10 plus récents
        }
      },
      $inc: { totalComments: 1 }
    }
  );
}

// Lire tous les commentaires si nécessaire
async function getAllComments(articleId) {
  return await db.article_comments
    .find({ articleId: articleId })
    .sort({ createdAt: -1 })
    .toArray();
}
```

**Cas d'usage** :
```javascript
// ✅ E-commerce : Produit avec reviews récentes
{
  productId: "PROD123",
  name: "Wireless Headphones",
  price: 99.99,

  // Top 5 reviews les plus utiles
  topReviews: [/* 5 reviews */],
  totalReviews: 1547,
  averageRating: 4.2
}

// ✅ Réseau social : Profil avec posts récents
{
  userId: "user123",
  username: "alice",
  bio: "...",

  // 20 posts les plus récents
  recentPosts: [/* 20 posts */],
  totalPosts: 3421
}

// ✅ Système de tickets : Ticket avec dernières interactions
{
  ticketId: "TICK789",
  subject: "Login issue",
  status: "open",

  // 10 dernières interactions
  recentUpdates: [/* 10 updates */],
  totalUpdates: 156
}
```

**Avantages** :
- Chargement rapide des cas d'usage courants (90%)
- Document reste compact
- Accès aux données complètes quand nécessaire
- Pas de fragmentation

---

## ❌ DON'T : Imbriquer des Objets Volumineux Sans Limite

**Explication** : Les objets imbriqués profonds et volumineux sont aussi problématiques que les tableaux sans limite.

**Anti-patterns** :
```javascript
// ❌ Historique complet de modifications imbriqué
{
  _id: ObjectId("..."),
  documentId: "DOC123",
  currentVersion: { /* contenu actuel */ },

  // Historique complet de toutes les versions
  versionHistory: {
    v1: { content: "...", metadata: {...}, date: "..." },
    v2: { content: "...", metadata: {...}, date: "..." },
    // ... des centaines de versions
    v847: { content: "...", metadata: {...}, date: "..." }
  }
  // Document devient énorme au fil du temps
}

// ❌ Configuration avec tous les profils
{
  appId: "APP001",
  config: {
    production: { /* 1000 paramètres */ },
    staging: { /* 1000 paramètres */ },
    development: { /* 1000 paramètres */ },
    test: { /* 1000 paramètres */ },
    user1: { /* paramètres personnalisés */ },
    user2: { /* paramètres personnalisés */ },
    // ... des milliers d'utilisateurs
  }
}

// ❌ Cache volumineux imbriqué
{
  userId: "user123",
  profile: { /* données profil */ },

  // Cache de toutes les données utilisateur
  cache: {
    orders: [/* tous les orders */],
    wishlist: [/* tous les items */],
    browsing_history: [/* tout l'historique */],
    recommendations: [/* toutes les recommandations */]
  }
  // Document devient multi-megabytes
}
```

**Solution** :
```javascript
// ✅ Séparer les versions dans une collection dédiée
// Collection: documents
{
  _id: ObjectId("..."),
  documentId: "DOC123",
  currentVersion: 847,
  content: { /* contenu actuel */ },
  lastModified: ISODate("..."),
  // Référence à collection de versions
  versionsCollection: "document_versions"
}

// Collection: document_versions
{
  _id: ObjectId("..."),
  documentId: "DOC123",
  version: 1,
  content: { /* contenu de v1 */ },
  createdAt: ISODate("...")
}

// ✅ Séparer les configurations par environnement
// Collection: app_configs
{
  _id: ObjectId("..."),
  appId: "APP001",
  environment: "production",
  config: { /* paramètres production uniquement */ }
}
```

---

## ✅ DO : Utiliser GridFS pour les Fichiers Volumineux

**Explication** : GridFS est conçu spécifiquement pour stocker des fichiers dépassant 16 MB en les divisant en chunks.

**Quand utiliser GridFS** :
- Fichiers > 16 MB (obligation)
- Fichiers binaires volumineux (images, vidéos, PDFs)
- Besoin de streaming
- Fichiers qui changent rarement

**Usage** :
```javascript
// ✅ Stocker un fichier avec GridFS
import { GridFSBucket } from 'mongodb';

const bucket = new GridFSBucket(db, {
  bucketName: 'uploads'
});

// Upload d'un fichier
const uploadStream = bucket.openUploadStream('large_video.mp4', {
  metadata: {
    userId: 'user123',
    contentType: 'video/mp4',
    uploadedAt: new Date()
  }
});

fs.createReadStream('./large_video.mp4').pipe(uploadStream);

uploadStream.on('finish', () => {
  console.log('File uploaded:', uploadStream.id);
});

// Référencer le fichier dans votre document
await db.posts.insertOne({
  _id: ObjectId("..."),
  userId: 'user123',
  caption: "Check out this video!",
  videoFileId: uploadStream.id,  // Référence GridFS
  createdAt: new Date()
});
```

**Architecture GridFS** :
```javascript
// GridFS crée deux collections automatiquement

// Collection: uploads.files (métadonnées)
{
  _id: ObjectId("..."),
  length: 52428800,  // 50 MB
  chunkSize: 261120,  // 255 KB par chunk
  uploadDate: ISODate("..."),
  filename: "large_video.mp4",
  metadata: {
    userId: "user123",
    contentType: "video/mp4"
  }
}

// Collection: uploads.chunks (données)
{
  _id: ObjectId("..."),
  files_id: ObjectId("..."),  // Référence au fichier
  n: 0,  // Numéro du chunk (0, 1, 2, ...)
  data: BinData(0, "...base64...")  // 255 KB de données
}
// 200 chunks pour un fichier de 50 MB
```

**Ne PAS utiliser pour** :
```javascript
// ❌ Petits fichiers (< 100 KB)
// Overhead de GridFS non justifié
// Mieux : stocker directement en BinData

// ❌ Données fréquemment modifiées
// GridFS pas optimisé pour les updates partielles

// ❌ Besoin d'accès par champ
// GridFS stocke les données en binaire opaque
```

---

## ✅ DO : Monitorer la Taille des Documents

**Explication** : Mettre en place une surveillance proactive de la taille des documents pour détecter les croissances anormales.

**Monitoring** :
```javascript
// ✅ Analyser la distribution des tailles
db.collection.aggregate([
  {
    $project: {
      sizeInBytes: { $bsonSize: "$$ROOT" },
      _id: 1,
      // Autres champs pertinents pour l'analyse
    }
  },
  {
    $bucket: {
      groupBy: "$sizeInBytes",
      boundaries: [
        0,
        10240,      // 10 KB
        102400,     // 100 KB
        524288,     // 500 KB
        1048576,    // 1 MB
        5242880,    // 5 MB
        16777216    // 16 MB
      ],
      default: "Over 16MB",
      output: {
        count: { $sum: 1 },
        avgSize: { $avg: "$sizeInBytes" },
        maxSize: { $max: "$sizeInBytes" },
        examples: { $push: "$_id" }
      }
    }
  }
]);

// Résultat
[
  { _id: 0, count: 850000, avgSize: 5120, maxSize: 10239 },       // < 10 KB
  { _id: 10240, count: 145000, avgSize: 45678, maxSize: 102399 }, // 10-100 KB
  { _id: 102400, count: 4500, avgSize: 256789, maxSize: 524287 }, // 100-500 KB
  { _id: 524288, count: 450, avgSize: 750123, maxSize: 1048575 }, // 500 KB-1 MB
  { _id: 1048576, count: 45, avgSize: 2500000, maxSize: 5242879 },// 1-5 MB
  { _id: 5242880, count: 5, avgSize: 8900000, maxSize: 15000000 } // 5-16 MB ⚠️
]
```

**Alertes proactives** :
```javascript
// ✅ Script de surveillance quotidien
async function checkDocumentSizes() {
  const largeDocs = await db.collection.find({
    $expr: {
      $gt: [{ $bsonSize: "$$ROOT" }, 1048576]  // > 1 MB
    }
  }).limit(100).toArray();

  if (largeDocs.length > 0) {
    console.warn(`Found ${largeDocs.length} documents over 1 MB`);

    // Analyser les causes
    for (const doc of largeDocs) {
      const size = JSON.stringify(doc).length;
      console.warn(`Document ${doc._id}: ${(size/1024/1024).toFixed(2)} MB`);

      // Identifier les champs volumineux
      for (const [key, value] of Object.entries(doc)) {
        const fieldSize = JSON.stringify(value).length;
        if (fieldSize > 100000) {  // > 100 KB
          console.warn(`  - Field "${key}": ${(fieldSize/1024).toFixed(2)} KB`);
        }
      }
    }

    // Envoyer alerte
    await sendAlert({
      type: 'LARGE_DOCUMENTS',
      count: largeDocs.length,
      collection: 'collection_name'
    });
  }
}
```

**Métriques à suivre** :
```javascript
// Dashboard de monitoring
const metrics = {
  totalDocuments: 1000000,
  averageSize: 12345,         // bytes
  medianSize: 8192,           // bytes
  p95Size: 102400,            // 95th percentile
  p99Size: 524288,            // 99th percentile
  maxSize: 2097152,           // Document le plus gros

  // Distribution
  under10KB: 850000,          // 85%
  under100KB: 995000,         // 99.5%
  under1MB: 999550,           // 99.955%
  over1MB: 450,               // 0.045% ⚠️
  over5MB: 5,                 // 0.0005% 🚨

  // Croissance
  growthRate: {
    daily: 1.02,              // +2% par jour
    weekly: 1.15,             // +15% par semaine
    monthly: 1.75             // +75% par mois 🚨
  }
};
```

---

## ❌ DON'T : Ignorer les Signaux d'Alerte

**Explication** : Des symptômes spécifiques indiquent que vos documents deviennent trop volumineux.

**Signaux d'alerte** :

### 1. Performance Dégradée
```javascript
// Symptômes :
// - Requêtes qui prenaient 10ms prennent maintenant 500ms
// - Timeout des requêtes simples
// - Latence réseau en hausse constante
// - CPU spike lors de parsing JSON

// ⚠️ Investiguer la taille des documents
```

### 2. Erreurs de Fragmentation
```javascript
// Logs MongoDB
"Moved document to new location due to growth"
"Document padding factor increased"

// ⚠️ Documents qui grossissent fréquemment
// Solution : Réviser la modélisation
```

### 3. Utilisation Mémoire Élevée
```javascript
// Symptômes :
// - WiredTiger cache constamment plein
// - Page faults en hausse
// - Swap mémoire utilisé
// - OOM (Out of Memory) errors

// ⚠️ Documents trop gros pour le cache
```

### 4. Croissance Linéaire de la Base
```javascript
// Base de données
Jour 1  : 10 GB
Mois 1  : 50 GB
Mois 6  : 300 GB
An 1    : 600 GB

// Si croissance linéaire mais nombre de documents stable
// ⚠️ Documents grossissent en moyenne
```

### 5. Échecs d'Opérations
```javascript
// Erreurs fréquentes
"BSONObj size: X is invalid"
"Document too large"
"Allocation failed"

// 🚨 Documents approchent ou dépassent la limite
```

**Actions correctives** :
```javascript
// ✅ Plan d'action
1. Identifier les documents volumineux (query ci-dessus)
2. Analyser les champs responsables
3. Choisir le pattern approprié :
   - Bucket si séries temporelles
   - Subset si données partielles suffisent
   - Référencement si données indépendantes
   - GridFS si fichiers binaires
4. Migrer progressivement
5. Mettre en place monitoring continu
6. Documenter les limites et alertes
```

---

## ✅ DO : Référencer Plutôt qu'Imbriquer pour Données Volumineuses

**Explication** : Les données volumineuses ou indépendantes doivent être dans des documents séparés avec références.

**Critères de décision** :
```javascript
// ✅ Imbriquer (Embedded) si :
- Données < 10 KB
- Toujours accédées ensemble (ratio 1:1)
- Ne change pas indépendamment
- Pas réutilisé ailleurs
- Pas de risque de croissance

// ✅ Référencer si :
- Données > 10 KB
- Accès indépendant possible
- Change fréquemment de façon indépendante
- Partagé entre plusieurs documents
- Risque de croissance
```

**Exemple** :
```javascript
// ❌ Imbriquer des données volumineuses
{
  _id: ObjectId("..."),
  orderId: "ORD123",
  customerId: "CUST456",

  // Customer data imbriqué (peut être volumineux)
  customer: {
    _id: "CUST456",
    name: "Alice Smith",
    email: "alice@example.com",
    // 50 autres champs
    addresses: [/* 10 adresses */],
    paymentMethods: [/* 5 méthodes */],
    orderHistory: [/* 100 commandes */],  // 🚨 PROBLÈME
    preferences: { /* nombreux paramètres */ }
  },

  items: [/* 20 produits */],
  // Document devient > 1 MB
}

// ✅ Référencer pour garder documents compacts
{
  _id: ObjectId("..."),
  orderId: "ORD123",
  customerId: ObjectId("..."),  // Simple référence

  // Denormalisation minimale pour performance
  customerName: "Alice Smith",
  customerEmail: "alice@example.com",

  items: [/* 20 produits */]
  // Document reste < 50 KB
}

// Document customer séparé
{
  _id: ObjectId("..."),
  customerId: "CUST456",
  name: "Alice Smith",
  email: "alice@example.com",
  // Toutes les données customer
}
```

---

## ✅ DO : Implémenter la Pagination Stricte

**Explication** : Limiter systématiquement le nombre d'éléments retournés par les requêtes, surtout pour les tableaux imbriqués.

**Pagination des tableaux imbriqués** :
```javascript
// ✅ Slice pour limiter les éléments retournés
db.articles.find(
  { _id: articleId },
  {
    title: 1,
    content: 1,
    // Retourner seulement les 10 premiers commentaires
    comments: { $slice: 10 }
  }
);

// ✅ Slice avec offset (pagination)
db.articles.find(
  { _id: articleId },
  {
    title: 1,
    // Sauter les 20 premiers, retourner les 10 suivants
    comments: { $slice: [20, 10] }
  }
);

// ✅ Utiliser $slice dans les agrégations
db.articles.aggregate([
  { $match: { _id: articleId } },
  {
    $project: {
      title: 1,
      content: 1,
      // Commentaires récents seulement
      recentComments: { $slice: ["$comments", -10] }
    }
  }
]);
```

**Pagination avec métadonnées** :
```javascript
// ✅ Réponse API paginée avec métadonnées
async function getComments(articleId, page = 1, pageSize = 20) {
  const skip = (page - 1) * pageSize;

  // Compter le total
  const total = await db.article_comments.countDocuments({
    articleId: articleId
  });

  // Récupérer la page
  const comments = await db.article_comments
    .find({ articleId: articleId })
    .sort({ createdAt: -1 })
    .skip(skip)
    .limit(pageSize)
    .toArray();

  return {
    data: comments,
    pagination: {
      page: page,
      pageSize: pageSize,
      total: total,
      totalPages: Math.ceil(total / pageSize),
      hasNext: skip + pageSize < total,
      hasPrev: page > 1
    }
  };
}
```

---

## ❌ DON'T : Charger des Documents Entiers Quand Seule une Partie est Nécessaire

**Explication** : Toujours utiliser des projections pour ne récupérer que les champs nécessaires, surtout avec des documents volumineux.

**Anti-pattern** :
```javascript
// ❌ Charger le document entier (2 MB)
const user = await db.users.findOne({ _id: userId });
// Puis utiliser seulement
const email = user.email;

// Transfert réseau : 2 MB
// Parsing JSON : 100ms
// Pour utiliser 50 bytes (l'email)
```

**Solution** :
```javascript
// ✅ Projection minimale
const user = await db.users.findOne(
  { _id: userId },
  { projection: { email: 1, _id: 0 } }
);

// Transfert réseau : 50 bytes
// Parsing JSON : <1ms
// = 40,000x plus efficace
```

**Projections avancées** :
```javascript
// ✅ Exclure les champs volumineux
const article = await db.articles.findOne(
  { _id: articleId },
  {
    projection: {
      // Inclure les champs légers
      title: 1,
      author: 1,
      publishedAt: 1,
      summary: 1,

      // Exclure les champs volumineux
      content: 0,           // Contenu complet (peut être 100 KB+)
      comments: 0,          // Tous les commentaires (peut être 1 MB+)
      versionHistory: 0     // Historique (peut être 500 KB+)
    }
  }
);

// Charger le contenu complet seulement si nécessaire
if (needFullContent) {
  const fullArticle = await db.articles.findOne(
    { _id: articleId },
    { projection: { content: 1, _id: 0 } }
  );
  article.content = fullArticle.content;
}
```

---

## Patterns Avancés

### ✅ DO : Pattern Outlier pour Cas Exceptionnels

**Explication** : Traiter différemment les documents exceptionnellement volumineux des documents normaux.

**Pattern** :
```javascript
// ✅ Détecter et marquer les outliers
{
  _id: ObjectId("..."),
  userId: "celebrity_user",
  username: "famous_person",
  followerCount: 10000000,  // 10M followers

  // Flag outlier
  isOutlier: true,

  // Subset seulement pour les outliers
  topFollowers: [/* 100 top followers */],

  // Référence vers collection séparée
  followersCollection: "celebrity_followers"
}

// Documents normaux (< 10k followers)
{
  _id: ObjectId("..."),
  userId: "normal_user",
  username: "alice",
  followerCount: 523,

  // Embedded directement (pas un outlier)
  followers: [/* 523 followers */]
}
```

**Implémentation** :
```javascript
// ✅ Logique conditionnelle basée sur outlier
async function getFollowers(userId) {
  const user = await db.users.findOne({ _id: userId });

  if (user.isOutlier) {
    // Cas outlier : récupérer depuis collection séparée
    return await db.celebrity_followers
      .find({ userId: userId })
      .limit(100)
      .toArray();
  } else {
    // Cas normal : embedded directement
    return user.followers || [];
  }
}

// Migration automatique vers outlier
async function checkAndMigrateOutlier(userId) {
  const user = await db.users.findOne({ _id: userId });

  if (!user.isOutlier && user.followers.length > 10000) {
    // Migrer vers outlier
    await db.celebrity_followers.insertMany(
      user.followers.map(f => ({
        userId: userId,
        followerId: f
      }))
    );

    await db.users.updateOne(
      { _id: userId },
      {
        $set: {
          isOutlier: true,
          topFollowers: user.followers.slice(0, 100)
        },
        $unset: { followers: "" }
      }
    );
  }
}
```

---

### ✅ DO : Pattern Extended Reference pour Dénormalisation Contrôlée

**Explication** : Dupliquer sélectivement quelques champs fréquemment accédés plutôt que tout imbriquer.

**Pattern** :
```javascript
// ✅ Extended reference : référence + champs critiques
{
  _id: ObjectId("..."),
  orderId: "ORD123",

  // Référence
  customerId: ObjectId("..."),

  // Champs dénormalisés (extended reference)
  customerName: "Alice Smith",
  customerEmail: "alice@example.com",
  customerTier: "premium",

  // Pas toutes les données customer (50+ champs)
  // Juste ce qui est fréquemment nécessaire

  items: [/* produits */],
  total: 150.00
}
```

**Bénéfices** :
- Performance : Pas besoin de jointure pour 90% des cas
- Taille : Document reste compact
- Maintenance : Mise à jour synchronisée si nécessaire

---

## Checklist Documents Volumineux

### Prévention
- [ ] Documents cibles < 100 KB
- [ ] Limite d'alerte à 1 MB configurée
- [ ] Pas de tableaux sans limite de taille
- [ ] Pas d'objets imbriqués profonds volumineux
- [ ] GridFS utilisé pour fichiers > 16 MB
- [ ] Patterns appropriés (Bucket, Subset, Référence)

### Monitoring
- [ ] Surveillance taille documents en place
- [ ] Alertes sur documents > 1 MB
- [ ] Métriques de distribution de taille
- [ ] Tracking de croissance dans le temps
- [ ] Analyse des champs volumineux

### Requêtes
- [ ] Projections utilisées systématiquement
- [ ] Pagination stricte implémentée
- [ ] Slice pour tableaux imbriqués
- [ ] Pas de chargement complet inutile

### Architecture
- [ ] Séparation données volumineuses
- [ ] Collections dédiées si nécessaire
- [ ] Références plutôt qu'embedding
- [ ] Pattern Outlier pour cas exceptionnels

---

## Tableau de Décision : Imbriquer vs Référencer

| Critère | Imbriquer | Référencer |
|---------|-----------|------------|
| **Taille totale** | < 10 KB | > 10 KB |
| **Fréquence d'accès ensemble** | Toujours (>95%) | Variable (<50%) |
| **Indépendance** | Totalement dépendant | Peut exister seul |
| **Réutilisation** | Unique à ce document | Partagé/réutilisé |
| **Fréquence de modification** | Rare ou synchrone | Fréquente et indépendante |
| **Croissance** | Fixe ou limitée | Potentiellement illimitée |
| **Performance critique** | Oui (1 requête) | Non (2+ requêtes OK) |

---

## Conclusion

La gestion de la taille des documents est un équilibre entre :

- **Performance** : Documents compacts = rapidité
- **Flexibilité** : Schéma adaptable sans migration lourde
- **Maintenabilité** : Code simple et prévisible
- **Scalabilité** : Croissance supportable à long terme

**Règles d'or** :
1. **Limite pratique** : Viser < 100 KB, alerter à 1 MB
2. **Pas de croissance illimitée** : Utiliser Bucket, Subset ou Références
3. **GridFS** : Pour fichiers > 16 MB
4. **Monitoring proactif** : Détecter les problèmes tôt
5. **Projections** : Ne charger que le nécessaire

Un document qui commence petit mais grossit sans contrôle devient un problème majeur en production. La prévention et le monitoring sont essentiels.

---


⏭️ [Éviter les collections excessives](/21-bonnes-pratiques-anti-patterns/05-eviter-collections-excessives.md)
