🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.1 Applications Web et Mobiles

## Introduction

Les applications web et mobiles représentent le cas d'usage le plus courant pour MongoDB. Ces applications se caractérisent par :

- **Schémas évolutifs** : Les besoins fonctionnels changent rapidement
- **Données hétérogènes** : Profils utilisateurs, contenus, interactions variés
- **Scalabilité variable** : De quelques utilisateurs à des millions
- **Exigences de latence** : Réponses rapides attendues (< 100ms)
- **Multi-plateforme** : Web, iOS, Android, APIs

MongoDB excelle dans ce contexte grâce à sa flexibilité de schéma, sa performance en lecture/écriture, et sa capacité à gérer des données complexes sans jointures multiples.

## Architecture de référence

### Stack technologique typique

```
┌─────────────────────────────────────────────────────┐
│              Load Balancer / CDN                    │
│         (NGINX, Cloudflare, CloudFront)             │
└────────────────────┬────────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼─────┐   ┌─────▼────┐   ┌──────▼───┐
│  Web App │   │  Web App │   │  Web App │
│ (Node.js)│   │ (Node.js)│   │ (Node.js)│
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
     │    ┌─────────┴────────┐     │
     │    │                  │     │
     └────┤   Application    ├─────┘
          │   Load Balancer  │
          └────────┬─────────┘
                   │
          ┌────────┴──────────┐
          │                   │
     ┌────▼────┐         ┌────▼─────┐
     │  Redis  │         │  Redis   │
     │ (Cache) │         │ (Session)│
     └─────────┘         └──────────┘
          │                   │
          └────────┬──────────┘
                   │
     ┌─────────────▼───────────────┐
     │     MongoDB Replica Set     │
     │  ┌─────────────────────┐    │
     │  │      Primary        │    │
     │  └──────────┬──────────┘    │
     │             │               │
     │   ┌─────────┴─────────┐     │
     │   │                   │     │
     │ ┌─▼────────┐   ┌──────▼───┐ │
     │ │Secondary1│   │Secondary2│ │
     │ └──────────┘   └──────────┘ │
     └─────────────────────────────┘
```

### Composants clés

#### 1. Couche Application
**Technologie recommandée :** Node.js, Python (FastAPI/Django), Java (Spring Boot)

**Justification :**
- Node.js : Excellente intégration avec MongoDB (drivers officiels)
- Event-driven architecture naturelle pour MongoDB
- Communauté large et ecosystem mature

**Configuration type :**
```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(process.env.MONGODB_URI, {
  maxPoolSize: 50,           // Pool de connexions
  minPoolSize: 10,
  maxIdleTimeMS: 30000,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  compressors: ['zstd', 'zlib'],  // Compression réseau
  readPreference: 'primaryPreferred',
  retryWrites: true,
  retryReads: true,
  w: 'majority',             // Write concern pour durabilité
  journal: true
});
```

#### 2. Couche Cache
**Technologie recommandée :** Redis ou Memcached

**Stratégie de caching :**
```
┌─────────────┐
│   Request   │
└──────┬──────┘
       │
   ┌───▼────┐
   │ Cache? │──Yes──► Return cached data
   └───┬────┘
       │ No
   ┌───▼────────┐
   │  MongoDB   │
   └───┬────────┘
       │
   ┌───▼────────┐
   │ Store cache│
   └───┬────────┘
       │
   ┌───▼────────┐
   │   Return   │
   └────────────┘
```

**Décisions de caching :**
- **Cache aside** : L'application gère le cache explicitement
- **TTL adaptatif** : Selon la fréquence de modification des données
- **Cache warming** : Pré-chargement des données critiques au démarrage

#### 3. Couche Données
**Configuration MongoDB :** Replica Set 3 nœuds minimum

**Justification :**
- **Haute disponibilité** : Failover automatique en cas de panne
- **Performance en lecture** : Distribution des lectures sur secondaries
- **Backup continu** : Oplog pour point-in-time recovery
- **Zéro downtime** : Maintenance sans interruption

## Modélisation des données

### 1. Profils utilisateurs

#### Modèle de base

```javascript
// Collection: users
{
  _id: ObjectId("..."),
  email: "user@example.com",
  username: "john_doe",
  passwordHash: "...",  // bcrypt ou argon2

  // Informations personnelles
  profile: {
    firstName: "John",
    lastName: "Doe",
    avatar: "https://cdn.example.com/avatars/john_doe.jpg",
    bio: "Full-stack developer",
    location: {
      city: "Paris",
      country: "France",
      coordinates: {
        type: "Point",
        coordinates: [2.3522, 48.8566]  // [longitude, latitude]
      }
    }
  },

  // Préférences
  preferences: {
    language: "fr",
    timezone: "Europe/Paris",
    theme: "dark",
    notifications: {
      email: true,
      push: true,
      sms: false
    }
  },

  // Métadonnées
  createdAt: ISODate("2024-01-15T10:30:00Z"),
  updatedAt: ISODate("2024-12-09T08:15:00Z"),
  lastLoginAt: ISODate("2024-12-09T08:00:00Z"),

  // Sécurité
  roles: ["user", "premium"],
  accountStatus: "active",  // active, suspended, deleted
  emailVerified: true,
  phoneVerified: false,

  // Données de session
  sessions: [
    {
      sessionId: "sess_abc123...",
      deviceInfo: {
        type: "mobile",
        os: "iOS",
        browser: "Safari"
      },
      ipAddress: "192.168.1.1",
      createdAt: ISODate("2024-12-09T08:00:00Z"),
      expiresAt: ISODate("2024-12-16T08:00:00Z")
    }
  ],

  // Index pour recherche
  searchVector: ["john", "doe", "developer", "paris"]
}
```

#### Décisions de modélisation

**✅ Documents imbriqués pour :**
- **Profile** : Toujours récupéré avec l'utilisateur
- **Preferences** : Accès fréquent, rarement modifié individuellement
- **Sessions actives** : Nombre limité (< 10 typiquement)

**✅ Champs atomiques pour :**
- **Dates de modification** : Tracking des changements
- **Statuts** : Requêtes de filtrage fréquentes
- **Rôles** : Authorization checks rapides

**❌ Éviter de mettre en embedded :**
- Historique complet des connexions (volume croissant)
- Posts/contenus créés (relation 1-to-many potentiellement grande)
- Followers/following (mieux géré dans collection séparée)

#### Index recommandés

```javascript
// Index uniques
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ username: 1 }, { unique: true });

// Index de recherche
db.users.createIndex({ "profile.firstName": 1, "profile.lastName": 1 });
db.users.createIndex({ "profile.location.city": 1 });

// Index géospatial
db.users.createIndex({ "profile.location.coordinates": "2dsphere" });

// Index de filtrage
db.users.createIndex({ accountStatus: 1, roles: 1 });
db.users.createIndex({ createdAt: -1 });

// Index pour authentification
db.users.createIndex({ email: 1, passwordHash: 1 });

// Index composite pour pagination
db.users.createIndex({ accountStatus: 1, createdAt: -1 });

// Index TTL pour sessions expirées
db.users.createIndex(
  { "sessions.expiresAt": 1 },
  { expireAfterSeconds: 0 }
);
```

### 2. Gestion des sessions

#### Option A : Sessions dans MongoDB

```javascript
// Collection: sessions
{
  _id: "sess_abc123def456...",  // Session ID
  userId: ObjectId("..."),

  data: {
    // Données de session sérialisées
    cart: [...],
    preferences: {...},
    tempData: {...}
  },

  device: {
    type: "mobile",
    os: "Android",
    appVersion: "2.4.1",
    deviceId: "device_unique_id"
  },

  security: {
    ipAddress: "192.168.1.1",
    userAgent: "Mozilla/5.0...",
    lastActivity: ISODate("2024-12-09T08:15:00Z")
  },

  createdAt: ISODate("2024-12-09T08:00:00Z"),
  expiresAt: ISODate("2024-12-09T20:00:00Z")
}

// Index TTL pour expiration automatique
db.sessions.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);

// Index pour recherche par utilisateur
db.sessions.createIndex({ userId: 1, lastActivity: -1 });
```

**Avantages :**
- Persistance garantie
- Requêtes complexes possibles (sessions actives d'un user)
- Synchronisation entre serveurs automatique

**Inconvénients :**
- Latence légèrement supérieure à Redis
- Plus de charge sur MongoDB

#### Option B : Sessions dans Redis

```javascript
// Structure Redis
Key: sess:abc123def456
Value: JSON.stringify({
  userId: "...",
  data: {...},
  device: {...},
  createdAt: "..."
})
TTL: 43200  // 12 heures en secondes

// Code Node.js
const session = await redis.get(`sess:${sessionId}`);
if (!session) {
  // Session expirée ou invalide
  return null;
}

const sessionData = JSON.parse(session);
```

**Avantages :**
- Latence ultra-faible (< 1ms)
- Charge réduite sur MongoDB
- Scalabilité horizontale simple

**Inconvénients :**
- Pas de requêtes complexes
- Dépendance additionnelle
- Persistence optionnelle (risque de perte)

#### Recommandation : Architecture hybride

```javascript
// Sessions courtes (< 1h) : Redis
// Sessions persistantes : MongoDB
// Synchronisation périodique Redis → MongoDB

class SessionManager {
  async createSession(userId, deviceInfo) {
    const sessionId = generateSecureId();
    const sessionData = {
      userId,
      device: deviceInfo,
      createdAt: new Date(),
      expiresAt: new Date(Date.now() + 12 * 3600000)
    };

    // Store in Redis for fast access
    await redis.setex(
      `sess:${sessionId}`,
      43200,
      JSON.stringify(sessionData)
    );

    // Store in MongoDB for persistence
    await db.collection('sessions').insertOne({
      _id: sessionId,
      ...sessionData
    });

    return sessionId;
  }

  async getSession(sessionId) {
    // Try Redis first
    let session = await redis.get(`sess:${sessionId}`);

    if (session) {
      return JSON.parse(session);
    }

    // Fallback to MongoDB
    session = await db.collection('sessions')
      .findOne({ _id: sessionId });

    if (session) {
      // Refresh Redis cache
      await redis.setex(
        `sess:${sessionId}`,
        43200,
        JSON.stringify(session)
      );
    }

    return session;
  }
}
```

### 3. Contenu et posts

#### Modèle pour application sociale

```javascript
// Collection: posts
{
  _id: ObjectId("..."),
  authorId: ObjectId("..."),

  // Contenu
  content: {
    text: "Check out this amazing sunset! 🌅",
    media: [
      {
        type: "image",
        url: "https://cdn.example.com/images/sunset.jpg",
        thumbnail: "https://cdn.example.com/thumbs/sunset.jpg",
        metadata: {
          width: 1920,
          height: 1080,
          size: 245678,
          format: "jpg"
        }
      }
    ],
    hashtags: ["sunset", "nature", "photography"],
    mentions: [ObjectId("..."), ObjectId("...")]
  },

  // Statistiques dénormalisées (pour performance)
  stats: {
    likes: 142,
    comments: 23,
    shares: 8,
    views: 1547
  },

  // Localisation
  location: {
    name: "Santa Monica Beach",
    coordinates: {
      type: "Point",
      coordinates: [-118.4912, 34.0095]
    }
  },

  // Métadonnées
  visibility: "public",  // public, friends, private
  status: "published",   // draft, published, archived
  createdAt: ISODate("2024-12-09T18:30:00Z"),
  updatedAt: ISODate("2024-12-09T18:30:00Z"),
  publishedAt: ISODate("2024-12-09T18:30:00Z"),

  // Données pré-calculées pour newsfeed
  feedScore: 87.5,  // Algorithme de ranking

  // Version du schéma (pour migrations)
  schemaVersion: 2
}
```

#### Index optimisés pour newsfeed

```javascript
// Feed principal (posts récents d'utilisateurs suivis)
db.posts.createIndex({
  authorId: 1,
  status: 1,
  publishedAt: -1
});

// Trending posts
db.posts.createIndex({
  status: 1,
  publishedAt: -1,
  "stats.likes": -1
});

// Recherche par hashtags
db.posts.createIndex({
  "content.hashtags": 1,
  publishedAt: -1
});

// Posts géolocalisés
db.posts.createIndex({
  "location.coordinates": "2dsphere",
  publishedAt: -1
});

// Recherche full-text
db.posts.createIndex({
  "content.text": "text",
  "content.hashtags": "text"
}, {
  weights: {
    "content.text": 2,
    "content.hashtags": 5
  }
});
```

### 4. Pattern : Extended Reference pour newsfeed

Au lieu de faire des jointures coûteuses, on dénormalise les informations essentielles :

```javascript
// Collection: posts (avec infos author embedded)
{
  _id: ObjectId("..."),
  authorId: ObjectId("..."),

  // ✅ Informations dénormalisées de l'auteur
  authorInfo: {
    username: "john_doe",
    avatar: "https://cdn.example.com/avatars/john_doe.jpg",
    verified: true,
    premiumMember: true
  },

  content: { /* ... */ },
  stats: { /* ... */ },
  publishedAt: ISODate("...")
}
```

**Avantages :**
- Une seule requête pour afficher le newsfeed
- Performance optimale (pas de jointure)
- Réduction drastique de la latence

**Gestion des mises à jour :**

```javascript
// Quand un utilisateur change son avatar
async function updateUserAvatar(userId, newAvatarUrl) {
  // 1. Mettre à jour l'utilisateur
  await db.collection('users').updateOne(
    { _id: userId },
    { $set: { "profile.avatar": newAvatarUrl } }
  );

  // 2. Mettre à jour tous les posts (async, en background)
  // Option A : Batch update
  await db.collection('posts').updateMany(
    { authorId: userId },
    { $set: { "authorInfo.avatar": newAvatarUrl } }
  );

  // Option B : Change Stream (plus élégant)
  // Un worker écoute les changements sur users
  // et met à jour posts automatiquement
}
```

### 5. Likes et interactions

#### Option A : Compteurs dans le post (recommandé)

```javascript
// Collection: posts
{
  _id: ObjectId("..."),
  stats: {
    likes: 142,        // ✅ Compteur dénormalisé
    comments: 23,
    shares: 8
  }
}

// Collection séparée: post_likes
{
  _id: ObjectId("..."),
  postId: ObjectId("..."),
  userId: ObjectId("..."),
  createdAt: ISODate("...")
}

// Index unique pour éviter les doublons
db.post_likes.createIndex(
  { postId: 1, userId: 1 },
  { unique: true }
);

// Index pour récupérer likes d'un utilisateur
db.post_likes.createIndex({ userId: 1, createdAt: -1 });
```

**Opération de like :**

```javascript
async function likePost(postId, userId) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // 1. Insérer le like
      await db.collection('post_likes').insertOne(
        { postId, userId, createdAt: new Date() },
        { session }
      );

      // 2. Incrémenter le compteur
      await db.collection('posts').updateOne(
        { _id: postId },
        { $inc: { "stats.likes": 1 } },
        { session }
      );
    });

    return { success: true };
  } catch (error) {
    if (error.code === 11000) {
      // Duplicate key = déjà liké
      return { success: false, message: "Already liked" };
    }
    throw error;
  } finally {
    await session.endSession();
  }
}
```

#### Option B : Array de likes dans le post (petite échelle)

```javascript
// ⚠️ Seulement si < 1000 likes attendus
{
  _id: ObjectId("..."),
  likes: [
    { userId: ObjectId("..."), createdAt: ISODate("...") },
    { userId: ObjectId("..."), createdAt: ISODate("...") }
    // ...
  ],
  likesCount: 142  // Compteur pour éviter $size
}

// Opération de like
db.posts.updateOne(
  {
    _id: postId,
    "likes.userId": { $ne: userId }  // Pas déjà liké
  },
  {
    $push: { likes: { userId, createdAt: new Date() } },
    $inc: { likesCount: 1 }
  }
);
```

**Limite :** Problème de taille de document si trop de likes (16 Mo max).

### 6. Commentaires

#### Modèle hiérarchique avec références

```javascript
// Collection: comments
{
  _id: ObjectId("..."),
  postId: ObjectId("..."),
  authorId: ObjectId("..."),

  // Dénormalisation de l'auteur
  authorInfo: {
    username: "jane_doe",
    avatar: "https://cdn.example.com/avatars/jane_doe.jpg"
  },

  // Contenu
  text: "Beautiful sunset! Where is this?",

  // Hiérarchie (pour nested comments)
  parentCommentId: null,  // null si commentaire racine
  depth: 0,               // 0 = racine, 1 = réponse, 2 = réponse à réponse

  // Statistiques
  stats: {
    likes: 5,
    replies: 2
  },

  // Métadonnées
  createdAt: ISODate("2024-12-09T18:45:00Z"),
  updatedAt: ISODate("2024-12-09T18:45:00Z"),
  edited: false,
  deleted: false  // Soft delete pour préserver les réponses
}

// Index optimisés
db.comments.createIndex({ postId: 1, createdAt: 1 });
db.comments.createIndex({ postId: 1, parentCommentId: 1 });
db.comments.createIndex({ authorId: 1, createdAt: -1 });
```

**Récupération des commentaires avec agrégation :**

```javascript
// Récupérer commentaires racine + leurs réponses
db.comments.aggregate([
  // 1. Filtrer par post
  { $match: { postId: ObjectId("..."), deleted: false } },

  // 2. Trier par date
  { $sort: { createdAt: 1 } },

  // 3. Regrouper réponses sous commentaires parents
  {
    $group: {
      _id: "$parentCommentId",
      comments: { $push: "$$ROOT" }
    }
  },

  // 4. Restructurer
  {
    $project: {
      _id: 0,
      parentId: "$_id",
      comments: 1
    }
  }
]);
```

## Architecture pour applications mobiles

### Spécificités mobiles

```
┌──────────────┐         ┌──────────────┐
│   iOS App    │         │ Android App  │
└──────┬───────┘         └──────┬───────┘
       │                        │
       │    ┌──────────────┐    │
       └────┤  REST API    ├────┘
            │  (GraphQL?)  │
            └──────┬───────┘
                   │
         ┌─────────┴──────────┐
         │                    │
    ┌────▼─────┐         ┌────▼─────┐
    │  Cache   │         │ Message  │
    │  Layer   │         │  Queue   │
    └──────────┘         └──────────┘
         │                    │
         └────────┬───────────┘
                  │
         ┌────────▼─────────┐
         │  MongoDB Atlas   │
         │  (Replica Set)   │
         └──────────────────┘
```

### 1. Synchronisation offline

#### Pattern : Sync Queue

```javascript
// Collection: sync_queue (sur device local)
{
  _id: "local_id_123",
  action: "create",  // create, update, delete
  collection: "posts",
  data: { /* données du post */ },
  timestamp: ISODate("2024-12-09T10:00:00Z"),
  synced: false,
  retryCount: 0,
  error: null
}

// Collection: sync_state (état de sync globale)
{
  userId: ObjectId("..."),
  lastSyncAt: ISODate("2024-12-09T09:30:00Z"),
  collections: {
    posts: { lastSyncToken: "token_abc123" },
    comments: { lastSyncToken: "token_def456" }
  }
}
```

**Logique de synchronisation :**

```javascript
class MobileSyncManager {
  async syncToServer() {
    const pendingActions = await localDB.sync_queue
      .find({ synced: false })
      .sort({ timestamp: 1 })
      .toArray();

    for (const action of pendingActions) {
      try {
        // Envoyer au serveur
        const result = await api.sync({
          action: action.action,
          collection: action.collection,
          data: action.data
        });

        // Marquer comme synchronisé
        await localDB.sync_queue.updateOne(
          { _id: action._id },
          {
            $set: {
              synced: true,
              serverId: result.serverId,
              syncedAt: new Date()
            }
          }
        );
      } catch (error) {
        // Incrémenter retry count
        await localDB.sync_queue.updateOne(
          { _id: action._id },
          {
            $inc: { retryCount: 1 },
            $set: { error: error.message }
          }
        );
      }
    }
  }

  async syncFromServer() {
    const syncState = await localDB.sync_state
      .findOne({ userId: currentUserId });

    // Récupérer changements depuis dernière sync
    const changes = await api.getChanges({
      since: syncState.lastSyncAt,
      collections: ['posts', 'comments', 'users']
    });

    // Appliquer les changements localement
    for (const change of changes) {
      await this.applyChange(change);
    }

    // Mettre à jour sync state
    await localDB.sync_state.updateOne(
      { userId: currentUserId },
      { $set: { lastSyncAt: new Date() } }
    );
  }
}
```

### 2. Gestion des conflits

#### Stratégie : Last Write Wins (LWW)

```javascript
// Chaque document a un vecteur de version
{
  _id: ObjectId("..."),
  content: "...",
  version: 5,
  updatedAt: ISODate("2024-12-09T10:30:00Z"),
  updatedBy: ObjectId("...")
}

// Lors d'une mise à jour
async function updateWithConflictResolution(docId, newContent, clientVersion) {
  const doc = await db.collection('posts').findOne({ _id: docId });

  if (doc.version > clientVersion) {
    // Conflit détecté
    return {
      success: false,
      conflict: true,
      serverVersion: doc,
      message: "Document was modified by another user"
    };
  }

  // Pas de conflit, appliquer la mise à jour
  const result = await db.collection('posts').updateOne(
    { _id: docId, version: clientVersion },  // Optimistic locking
    {
      $set: { content: newContent, updatedAt: new Date() },
      $inc: { version: 1 }
    }
  );

  return { success: result.modifiedCount === 1 };
}
```

### 3. Optimisation de la bande passante

#### Pattern : Delta Sync

```javascript
// Au lieu d'envoyer tout le document, envoyer seulement les changements
{
  docId: ObjectId("..."),
  changes: [
    {
      op: "set",
      path: "profile.bio",
      value: "New bio"
    },
    {
      op: "inc",
      path: "stats.views",
      value: 1
    },
    {
      op: "push",
      path: "tags",
      value: "newTag"
    }
  ],
  timestamp: ISODate("...")
}

// Serveur applique les changements
async function applyDelta(docId, changes) {
  const updateOps = {};

  for (const change of changes) {
    switch (change.op) {
      case 'set':
        if (!updateOps.$set) updateOps.$set = {};
        updateOps.$set[change.path] = change.value;
        break;
      case 'inc':
        if (!updateOps.$inc) updateOps.$inc = {};
        updateOps.$inc[change.path] = change.value;
        break;
      case 'push':
        if (!updateOps.$push) updateOps.$push = {};
        updateOps.$push[change.path] = change.value;
        break;
    }
  }

  return db.collection('documents').updateOne(
    { _id: docId },
    updateOps
  );
}
```

### 4. Pagination et infinite scroll

#### Cursor-based pagination (recommandé)

```javascript
// ❌ Éviter : Offset-based (performances dégradées avec grand offset)
db.posts.find().skip(1000).limit(20)

// ✅ Utiliser : Cursor-based
async function getPostsFeed(lastPostId, limit = 20) {
  const query = lastPostId
    ? { _id: { $lt: ObjectId(lastPostId) } }
    : {};

  const posts = await db.collection('posts')
    .find(query)
    .sort({ _id: -1 })
    .limit(limit)
    .toArray();

  return {
    posts,
    nextCursor: posts.length > 0
      ? posts[posts.length - 1]._id.toString()
      : null,
    hasMore: posts.length === limit
  };
}

// Utilisation côté client
let cursor = null;
do {
  const response = await api.getFeed(cursor);
  displayPosts(response.posts);
  cursor = response.nextCursor;
} while (response.hasMore && userScrolling);
```

#### Optimisation avec index composé

```javascript
// Index pour newsfeed personnalisé
db.posts.createIndex({
  authorId: 1,
  _id: -1
});

// Requête optimisée
db.posts.find({
  authorId: { $in: followingUserIds },
  _id: { $lt: ObjectId(lastPostId) }
}).sort({ _id: -1 }).limit(20);
```

## Patterns de performance

### 1. Caching multi-niveaux

```javascript
class DataService {
  constructor(redis, mongo) {
    this.redis = redis;
    this.mongo = mongo;
    this.memoryCache = new Map();  // Cache L1
  }

  async getUser(userId) {
    // L1 : Memory cache (nanoseconds)
    if (this.memoryCache.has(userId)) {
      return this.memoryCache.get(userId);
    }

    // L2 : Redis (< 1ms)
    let user = await this.redis.get(`user:${userId}`);
    if (user) {
      user = JSON.parse(user);
      this.memoryCache.set(userId, user);
      return user;
    }

    // L3 : MongoDB (5-20ms)
    user = await this.mongo.collection('users')
      .findOne({ _id: ObjectId(userId) });

    if (user) {
      // Populate caches
      this.memoryCache.set(userId, user);
      await this.redis.setex(
        `user:${userId}`,
        300,  // 5 minutes
        JSON.stringify(user)
      );
    }

    return user;
  }

  async updateUser(userId, updates) {
    // Update MongoDB
    await this.mongo.collection('users').updateOne(
      { _id: ObjectId(userId) },
      { $set: updates }
    );

    // Invalidate caches
    this.memoryCache.delete(userId);
    await this.redis.del(`user:${userId}`);
  }
}
```

### 2. Read Preference pour séparation des charges

```javascript
// Configuration des read preferences
const readConfig = {
  userProfiles: 'primaryPreferred',    // Données critiques
  newsfeed: 'secondary',               // Données tolérantes
  analytics: 'secondary',              // Charge lourde sur secondary
  adminReports: 'secondary'            // Pas de fraîcheur critique
};

// Utilisation
async function getUserPosts(userId, useSecondary = true) {
  const collection = db.collection('posts');

  const readPref = useSecondary
    ? { readPreference: 'secondary' }
    : { readPreference: 'primary' };

  return collection
    .find({ authorId: userId }, readPref)
    .sort({ createdAt: -1 })
    .limit(20)
    .toArray();
}
```

### 3. Batching des écritures

```javascript
class WriteBuffer {
  constructor(db, options = {}) {
    this.db = db;
    this.buffer = [];
    this.maxSize = options.maxSize || 100;
    this.maxWait = options.maxWait || 1000;  // 1 seconde
    this.timer = null;
  }

  async addWrite(collection, operation, doc) {
    this.buffer.push({ collection, operation, doc });

    if (this.buffer.length >= this.maxSize) {
      await this.flush();
    } else if (!this.timer) {
      this.timer = setTimeout(() => this.flush(), this.maxWait);
    }
  }

  async flush() {
    if (this.buffer.length === 0) return;

    clearTimeout(this.timer);
    this.timer = null;

    const operations = this.buffer.splice(0);
    const grouped = this.groupByCollection(operations);

    for (const [collection, ops] of Object.entries(grouped)) {
      const bulkOps = ops.map(op => {
        switch (op.operation) {
          case 'insert':
            return { insertOne: { document: op.doc } };
          case 'update':
            return {
              updateOne: {
                filter: { _id: op.doc._id },
                update: { $set: op.doc.updates }
              }
            };
        }
      });

      await this.db.collection(collection).bulkWrite(bulkOps, {
        ordered: false  // Parallélisation
      });
    }
  }

  groupByCollection(operations) {
    return operations.reduce((acc, op) => {
      if (!acc[op.collection]) acc[op.collection] = [];
      acc[op.collection].push(op);
      return acc;
    }, {});
  }
}

// Utilisation pour analytics/tracking
const writeBuffer = new WriteBuffer(db);

app.post('/api/post/:id/view', async (req, res) => {
  // Incrémentation asynchrone sans attendre
  writeBuffer.addWrite('posts', 'update', {
    _id: ObjectId(req.params.id),
    updates: { $inc: { 'stats.views': 1 } }
  });

  res.json({ success: true });
});
```

## Monitoring et observabilité

### Métriques essentielles à suivre

```javascript
// Dashboard monitoring pour app web/mobile
const metrics = {
  // Performance MongoDB
  'mongodb.operations.read': {
    query: 'db.serverStatus().opcounters.query',
    threshold: 10000,  // ops/s
    alert: 'high_read_load'
  },

  'mongodb.operations.write': {
    query: 'db.serverStatus().opcounters.insert + update + delete',
    threshold: 5000,
    alert: 'high_write_load'
  },

  'mongodb.connections.current': {
    query: 'db.serverStatus().connections.current',
    threshold: 4000,  // 80% of maxPoolSize * instances
    alert: 'connection_pool_exhausted'
  },

  'mongodb.replication.lag': {
    query: 'rs.status().members.find(m => m.stateStr === "SECONDARY").optimeDate - primary.optimeDate',
    threshold: 10000,  // 10 secondes
    alert: 'replication_lag_high'
  },

  // Performance application
  'app.response.time.p95': {
    threshold: 200,  // ms
    alert: 'slow_response'
  },

  'app.error.rate': {
    threshold: 0.01,  // 1%
    alert: 'error_rate_high'
  },

  // Cache
  'redis.hit.rate': {
    threshold: 0.90,  // 90%
    alert: 'cache_miss_high'
  }
};
```

### Slow query profiling

```javascript
// Activer le profiler niveau 1 (slow queries > 100ms)
db.setProfilingLevel(1, { slowms: 100 });

// Analyser les slow queries
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  {
    $group: {
      _id: "$command.find",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
]);

// Script d'analyse automatique
async function analyzeSlowQueries() {
  const slowQueries = await db.system.profile.find({
    millis: { $gt: 100 }
  }).sort({ ts: -1 }).limit(100).toArray();

  const analysis = {
    missingIndexes: [],
    collectionScans: [],
    largeResults: []
  };

  for (const query of slowQueries) {
    // Détection de collection scan
    if (query.execStats?.executionStages?.stage === 'COLLSCAN') {
      analysis.collectionScans.push({
        collection: query.ns,
        filter: query.command.filter,
        time: query.millis
      });
    }

    // Détection de résultats volumineux
    if (query.nreturned > 1000) {
      analysis.largeResults.push({
        collection: query.ns,
        count: query.nreturned,
        time: query.millis
      });
    }
  }

  return analysis;
}
```

## Sécurité et authentification

### JWT avec refresh tokens

```javascript
// Collection: refresh_tokens
{
  _id: "token_abc123...",
  userId: ObjectId("..."),
  deviceId: "device_xyz...",
  tokenHash: "sha256_hash...",  // Hasher le token

  expiresAt: ISODate("2024-12-23T10:00:00Z"),  // 14 jours
  createdAt: ISODate("2024-12-09T10:00:00Z"),

  lastUsedAt: ISODate("2024-12-09T15:00:00Z"),
  lastIp: "192.168.1.1",

  revoked: false,
  revokedAt: null,
  revokedReason: null
}

// Index
db.refresh_tokens.createIndex({ tokenHash: 1 }, { unique: true });
db.refresh_tokens.createIndex({ userId: 1, deviceId: 1 });
db.refresh_tokens.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
```

**Workflow d'authentification :**

```javascript
class AuthService {
  async login(email, password) {
    // 1. Vérifier credentials
    const user = await db.collection('users')
      .findOne({ email, accountStatus: 'active' });

    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
      throw new Error('Invalid credentials');
    }

    // 2. Générer tokens
    const accessToken = jwt.sign(
      { userId: user._id, roles: user.roles },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }  // Short-lived
    );

    const refreshToken = crypto.randomBytes(32).toString('hex');
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    // 3. Stocker refresh token
    await db.collection('refresh_tokens').insertOne({
      _id: refreshToken,
      userId: user._id,
      deviceId: req.body.deviceId,
      tokenHash,
      expiresAt: new Date(Date.now() + 14 * 24 * 3600000),
      createdAt: new Date(),
      lastIp: req.ip
    });

    // 4. Update last login
    await db.collection('users').updateOne(
      { _id: user._id },
      { $set: { lastLoginAt: new Date() } }
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: 900,  // 15 minutes
      user: {
        id: user._id,
        username: user.username,
        email: user.email
      }
    };
  }

  async refreshAccessToken(refreshToken) {
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    const tokenDoc = await db.collection('refresh_tokens')
      .findOne({ tokenHash, revoked: false });

    if (!tokenDoc || tokenDoc.expiresAt < new Date()) {
      throw new Error('Invalid or expired refresh token');
    }

    // Générer nouveau access token
    const user = await db.collection('users')
      .findOne({ _id: tokenDoc.userId });

    const accessToken = jwt.sign(
      { userId: user._id, roles: user.roles },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }
    );

    // Update last used
    await db.collection('refresh_tokens').updateOne(
      { tokenHash },
      {
        $set: {
          lastUsedAt: new Date(),
          lastIp: req.ip
        }
      }
    );

    return { accessToken, expiresIn: 900 };
  }

  async logout(refreshToken) {
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    await db.collection('refresh_tokens').updateOne(
      { tokenHash },
      {
        $set: {
          revoked: true,
          revokedAt: new Date(),
          revokedReason: 'user_logout'
        }
      }
    );
  }
}
```

## Checklist de déploiement

### ✅ Configuration MongoDB

- [ ] Replica Set avec minimum 3 nœuds
- [ ] Write Concern `majority` pour données critiques
- [ ] Read Preference adapté aux cas d'usage
- [ ] Index optimisés pour toutes les requêtes fréquentes
- [ ] Profiler activé (niveau 1, slowms: 100)
- [ ] Connection pooling configuré (maxPoolSize: 50)
- [ ] Compression réseau activée (zstd)

### ✅ Sécurité

- [ ] Authentification SCRAM-SHA-256 activée
- [ ] TLS/SSL configuré pour toutes les connexions
- [ ] Principe du moindre privilège pour les utilisateurs
- [ ] IP whitelisting configuré
- [ ] Secrets dans variables d'environnement
- [ ] Audit logging activé pour actions sensibles

### ✅ Performance

- [ ] Cache Redis déployé
- [ ] CDN configuré pour assets statiques
- [ ] Pagination cursor-based implémentée
- [ ] Batching des écritures pour analytics
- [ ] Index couvrants pour requêtes fréquentes
- [ ] Query timeout configuré (30s max)

### ✅ Haute disponibilité

- [ ] Backup automatique quotidien
- [ ] Point-in-time recovery configuré
- [ ] Procédure de failover testée
- [ ] Multi-région si requis
- [ ] Health checks sur tous les nœuds
- [ ] Alerting configuré (PagerDuty/Opsgenie)

### ✅ Monitoring

- [ ] Métriques MongoDB dans dashboards
- [ ] Slow queries analysées quotidiennement
- [ ] Logs centralisés (ELK/Datadog)
- [ ] APM configuré (New Relic/Datadog)
- [ ] Alertes sur seuils critiques
- [ ] Runbooks pour incidents courants

## Conclusion

Les applications web et mobiles bénéficient particulièrement des forces de MongoDB :

**✅ Avantages démontrés :**
- Flexibilité de schéma pour itérations rapides
- Performance en lecture/écriture adaptée au OLTP
- Modélisation orientée document naturelle pour APIs
- Scaling horizontal avec sharding si nécessaire
- Ecosystem riche (drivers, ODM, outils)

**⚠️ Points d'attention :**
- Dénormalisation doit être maintenue (synchronisation)
- Index multiples impactent les performances d'écriture
- Transactions multi-documents ont un coût
- Backup et disaster recovery essentiels en production

**🎯 Recommandations finales :**
1. Commencer simple (Replica Set 3 nœuds)
2. Mesurer avant d'optimiser (profiling, explain)
3. Cacher agressivement (Redis, CDN)
4. Scaler horizontalement quand nécessaire (> 2TB)
5. Surveiller constamment (métriques, alertes)

L'architecture présentée supporte des applications allant de quelques milliers à plusieurs millions d'utilisateurs actifs, avec une évolution progressive selon les besoins réels.

---

**Références :**
- MongoDB Manual: Application Design
- MongoDB University: M220 (MongoDB for Developers)
- "Designing Data-Intensive Applications" - Martin Kleppmann

⏭️ [Gestion de contenu (CMS)](/20-cas-usage-architectures/02-gestion-contenu-cms.md)
