🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.2 Gestion de Contenu (CMS)

## Introduction

Les systèmes de gestion de contenu (CMS) représentent un cas d'usage idéal pour MongoDB en raison de la nature flexible et hiérarchique du contenu. Les CMS modernes doivent gérer :

- **Contenus polymorphes** : Articles, pages, vidéos, podcasts avec structures différentes
- **Hiérarchies complexes** : Catégories, tags, menus multiniveaux
- **Versioning** : Historique complet des modifications
- **Multi-langue** : Contenus traduits et localisés
- **Workflow éditorial** : Brouillon → Révision → Publication
- **Personnalisation** : Contenu dynamique selon l'utilisateur
- **Performance** : Temps de chargement rapides pour le SEO

MongoDB excelle dans ce contexte grâce à sa flexibilité de schéma, sa capacité à gérer des documents complexes, et ses fonctionnalités de requêtes puissantes.

## Architecture de référence

### Stack CMS moderne

```
┌─────────────────────────────────────────────────────┐
│                    CDN / Edge Cache                 │
│              (Cloudflare, Fastly, CDN77)            │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌────────▼─────────┐
│  Frontend Web  │       │  Admin Dashboard │
│  (Next.js SSG) │       │   (React/Vue)    │
└───────┬────────┘       └────────┬─────────┘
        │                         │
        └────────┬────────────────┘
                 │
        ┌────────▼────────┐
        │   API Gateway   │
        │  (REST/GraphQL) │
        └────────┬────────┘
                 │
     ┌───────────┼───────────┐
     │           │           │
┌────▼────┐ ┌───▼────┐ ┌─────▼───┐
│ Content │ │ Media  │ │  User   │
│ Service │ │Service │ │ Service │
└────┬────┘ └───┬────┘ └────┬────┘
     │          │           │
     └──────────┼───────────┘
                │
     ┌──────────┼──────────┐
     │          │          │
┌────▼──────┐   │   ┌──────▼──────┐
│  Redis    │   │   │ Elasticsearch
│  (Cache)  │   │   │  (Search)   │
└───────────┘   │   └─────────────┘
                │
      ┌─────────▼─────────┐
      │  MongoDB Atlas    │
      │   Replica Set     │
      │  ┌─────────────┐  │
      │  │   Primary   │  │
      │  └──────┬──────┘  │
      │         │         │
      │  ┌──────┴──────┐  │
      │  │             │  │
      │┌─▼──┐      ┌───▼─┐│
      ││Sec1│      │Sec2 ││
      │└────┘      └─────┘│
      └───────────────────┘
```

### Composants architecturaux

#### 1. Frontend découplé (Headless CMS)
**Justification :** Séparation backend/frontend pour flexibilité maximale

**Options courantes :**
- **Next.js** : SSG/SSR avec React, excellent pour SEO
- **Gatsby** : SSG optimisé, plugins riches
- **Nuxt.js** : Vue.js avec SSR/SSG
- **Astro** : Multi-framework, ultra-performant

**Pattern de déploiement :**
```javascript
// next.js avec ISR (Incremental Static Regeneration)
export async function getStaticProps({ params }) {
  const article = await fetchArticle(params.slug);

  return {
    props: { article },
    revalidate: 60  // Regenerate après 60 secondes
  };
}
```

#### 2. API Layer
**Technologies :** REST, GraphQL, ou hybride

**Exemple GraphQL Schema :**
```graphql
type Content {
  id: ID!
  type: ContentType!
  slug: String!
  title: String!
  body: JSON!
  status: ContentStatus!
  publishedAt: DateTime
  author: User!
  categories: [Category!]!
  tags: [Tag!]!
  metadata: ContentMetadata!
}

enum ContentType {
  ARTICLE
  PAGE
  VIDEO
  PODCAST
}

enum ContentStatus {
  DRAFT
  IN_REVIEW
  SCHEDULED
  PUBLISHED
  ARCHIVED
}
```

## Modélisation des données

### 1. Documents de contenu polymorphe

#### Pattern Attribute pour types variables

```javascript
// Collection: contents
{
  _id: ObjectId("..."),
  type: "article",  // article, page, video, podcast

  // Métadonnées communes
  slug: "introduction-mongodb-cms",
  status: "published",

  // Localization
  locale: "fr",
  translations: [
    { locale: "en", contentId: ObjectId("...") },
    { locale: "es", contentId: ObjectId("...") }
  ],

  // Informations éditoriales
  author: {
    id: ObjectId("..."),
    name: "John Doe",
    avatar: "https://cdn.example.com/authors/john.jpg"
  },

  // Dates
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z"),
  publishedAt: ISODate("2024-01-20T08:00:00Z"),
  scheduledAt: null,

  // SEO
  seo: {
    title: "Introduction à MongoDB pour CMS",
    description: "Guide complet pour utiliser MongoDB...",
    keywords: ["mongodb", "cms", "nosql"],
    ogImage: "https://cdn.example.com/og/article-123.jpg",
    canonicalUrl: "https://example.com/blog/introduction-mongodb-cms"
  },

  // Taxonomies
  categories: [
    { id: ObjectId("..."), name: "Tutorials", slug: "tutorials" }
  ],
  tags: ["mongodb", "database", "cms"],

  // Contenu spécifique au type
  data: {
    // Pour articles
    title: "Introduction à MongoDB pour CMS",
    excerpt: "MongoDB offre une flexibilité exceptionnelle...",
    featuredImage: {
      url: "https://cdn.example.com/images/featured-123.jpg",
      alt: "MongoDB Logo",
      width: 1200,
      height: 630
    },
    body: {
      // Structure de blocs (Gutenberg-like)
      blocks: [
        {
          id: "block_1",
          type: "paragraph",
          data: {
            text: "MongoDB est une base de données..."
          }
        },
        {
          id: "block_2",
          type: "heading",
          data: {
            level: 2,
            text: "Pourquoi MongoDB pour un CMS?"
          }
        },
        {
          id: "block_3",
          type: "image",
          data: {
            url: "https://cdn.example.com/images/diagram.jpg",
            caption: "Architecture d'un CMS moderne",
            width: 800,
            height: 600
          }
        },
        {
          id: "block_4",
          type: "code",
          data: {
            language: "javascript",
            code: "const article = await Content.findOne({ slug });"
          }
        }
      ]
    },
    readingTime: 8,  // minutes
    wordCount: 1500
  },

  // Statistiques
  stats: {
    views: 1247,
    likes: 89,
    comments: 23,
    shares: 45
  },

  // Versioning
  version: 5,
  versionHistory: [
    {
      version: 4,
      updatedAt: ISODate("2024-12-08T10:00:00Z"),
      updatedBy: ObjectId("..."),
      changes: "Updated introduction section"
    }
  ],

  // Index pour recherche
  searchText: "introduction mongodb cms guide complet...",

  // Schema version pour migrations
  schemaVersion: 3
}
```

#### Gestion des types de contenu différents

```javascript
// Collection: contents (différents types)

// Type: Article (structure ci-dessus)

// Type: Video
{
  _id: ObjectId("..."),
  type: "video",
  slug: "mongodb-tutorial-video",
  status: "published",

  data: {
    title: "MongoDB Tutorial - Getting Started",
    description: "Complete video tutorial...",
    thumbnail: "https://cdn.example.com/thumbs/video-456.jpg",

    // Spécifique vidéo
    videoUrl: "https://video.example.com/mongodb-tutorial.mp4",
    duration: 1845,  // secondes
    resolution: "1080p",
    chapters: [
      { time: 0, title: "Introduction" },
      { time: 120, title: "Installation" },
      { time: 300, title: "First Database" }
    ],
    transcript: {
      vtt: "https://cdn.example.com/transcripts/video-456.vtt",
      srt: "https://cdn.example.com/transcripts/video-456.srt"
    }
  }
}

// Type: Podcast
{
  _id: ObjectId("..."),
  type: "podcast",
  slug: "mongodb-podcast-ep-12",
  status: "published",

  data: {
    title: "Episode 12: MongoDB at Scale",
    description: "Discussion about scaling MongoDB...",
    coverArt: "https://cdn.example.com/podcast/cover-12.jpg",

    // Spécifique podcast
    audioUrl: "https://audio.example.com/episode-12.mp3",
    duration: 3600,
    episodeNumber: 12,
    season: 2,
    guests: [
      {
        name: "Jane Smith",
        bio: "MongoDB Expert",
        photo: "https://cdn.example.com/guests/jane.jpg"
      }
    ],
    showNotes: "In this episode we discuss...",
    chapters: [
      { time: 0, title: "Introduction" },
      { time: 180, title: "Main Topic" }
    ]
  }
}
```

#### Index optimisés pour contenus

```javascript
// Index pour recherche par slug (unique par locale)
db.contents.createIndex(
  { slug: 1, locale: 1 },
  { unique: true }
);

// Index pour listing et filtrage
db.contents.createIndex({
  status: 1,
  publishedAt: -1
});

// Index composite pour catégories
db.contents.createIndex({
  "categories.id": 1,
  status: 1,
  publishedAt: -1
});

// Index pour tags
db.contents.createIndex({
  tags: 1,
  status: 1,
  publishedAt: -1
});

// Index pour recherche full-text
db.contents.createIndex({
  searchText: "text",
  "data.title": "text",
  tags: "text"
}, {
  weights: {
    "data.title": 10,
    tags: 5,
    searchText: 1
  },
  default_language: "french",
  language_override: "locale"
});

// Index pour auteurs
db.contents.createIndex({
  "author.id": 1,
  status: 1,
  publishedAt: -1
});

// Index pour contenu programmé
db.contents.createIndex({
  status: 1,
  scheduledAt: 1
});

// Index pour analytics
db.contents.createIndex({
  type: 1,
  publishedAt: -1,
  "stats.views": -1
});
```

### 2. Hiérarchies et taxonomies

#### Catégories avec Materialized Path Pattern

```javascript
// Collection: categories
{
  _id: ObjectId("..."),
  name: "Backend Development",
  slug: "backend-development",

  // Materialized Path pour hiérarchie
  path: ",programming,backend-development,",

  // Facilite les requêtes d'ancêtres
  ancestors: [
    { id: ObjectId("..."), name: "Programming", slug: "programming" }
  ],

  // Métadonnées
  description: "All about backend development",
  icon: "server",
  color: "#3B82F6",

  // SEO
  seo: {
    title: "Backend Development Articles",
    description: "Explore our backend development content..."
  },

  // Position dans l'ordre d'affichage
  order: 10,

  // Statistiques
  contentCount: 45,  // Dénormalisé pour performance

  // Dates
  createdAt: ISODate("2024-01-01T00:00:00Z"),
  updatedAt: ISODate("2024-12-01T00:00:00Z")
}

// Index pour hiérarchies
db.categories.createIndex({ path: 1 });
db.categories.createIndex({ slug: 1 }, { unique: true });
db.categories.createIndex({ "ancestors.id": 1 });

// Requête : Tous les descendants de "Programming"
db.categories.find({
  path: { $regex: "^,programming," }
});

// Requête : Catégories racines
db.categories.find({
  ancestors: { $size: 0 }
});

// Requête : Enfants directs d'une catégorie
db.categories.find({
  "ancestors.id": ObjectId("parent_category_id"),
  "ancestors": { $size: 1 }  // 1 = enfant direct
});
```

#### Menus de navigation avec références

```javascript
// Collection: menus
{
  _id: ObjectId("..."),
  name: "Main Navigation",
  location: "header",  // header, footer, sidebar
  locale: "fr",

  items: [
    {
      id: "item_1",
      type: "link",
      label: "Accueil",
      url: "/",
      icon: "home",
      order: 1,

      // Sous-menu
      children: []
    },
    {
      id: "item_2",
      type: "category",
      label: "Articles",
      categoryId: ObjectId("..."),
      order: 2,

      children: [
        {
          id: "item_2_1",
          type: "category",
          label: "Tutorials",
          categoryId: ObjectId("..."),
          order: 1
        },
        {
          id: "item_2_2",
          type: "custom",
          label: "Tous les articles",
          url: "/articles",
          order: 2
        }
      ]
    },
    {
      id: "item_3",
      type: "page",
      label: "À propos",
      contentId: ObjectId("..."),  // Référence vers content
      order: 3
    }
  ],

  // Métadonnées
  isActive: true,
  createdAt: ISODate("2024-01-01T00:00:00Z"),
  updatedAt: ISODate("2024-12-05T00:00:00Z")
}

// Index
db.menus.createIndex({ location: 1, locale: 1, isActive: 1 });
```

### 3. Versioning et historique

#### Pattern : Versions séparées avec références

```javascript
// Collection: contents (version actuelle)
{
  _id: ObjectId("content_123"),
  slug: "article-slug",
  status: "published",
  version: 5,
  currentVersionId: ObjectId("version_5"),
  data: { /* contenu actuel */ },
  // ...
}

// Collection: content_versions (historique complet)
{
  _id: ObjectId("version_5"),
  contentId: ObjectId("content_123"),
  version: 5,

  // Snapshot complet du contenu
  snapshot: {
    slug: "article-slug",
    status: "published",
    data: { /* contenu à cette version */ },
    seo: { /* ... */ }
  },

  // Informations de version
  changes: {
    type: "major",  // major, minor, patch
    description: "Complete rewrite of introduction",
    fields: ["data.body.blocks[0]", "data.body.blocks[1]"]
  },

  // Métadonnées
  createdBy: {
    id: ObjectId("..."),
    name: "John Editor",
    role: "editor"
  },
  createdAt: ISODate("2024-12-09T14:30:00Z"),

  // Comparaison avec version précédente
  diff: {
    added: 450,     // caractères ajoutés
    removed: 120,   // caractères supprimés
    modified: 230   // caractères modifiés
  }
}

// Index
db.content_versions.createIndex({
  contentId: 1,
  version: -1
});

db.content_versions.createIndex({
  contentId: 1,
  createdAt: -1
});
```

#### Restauration d'une version antérieure

```javascript
async function restoreVersion(contentId, versionNumber) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // 1. Récupérer la version à restaurer
      const version = await db.collection('content_versions')
        .findOne({
          contentId: ObjectId(contentId),
          version: versionNumber
        });

      if (!version) {
        throw new Error('Version not found');
      }

      // 2. Créer une nouvelle version avec le snapshot
      const currentContent = await db.collection('contents')
        .findOne({ _id: ObjectId(contentId) });

      const newVersion = currentContent.version + 1;

      // 3. Sauvegarder version actuelle avant restoration
      await db.collection('content_versions').insertOne({
        contentId: ObjectId(contentId),
        version: newVersion - 1,
        snapshot: currentContent,
        changes: {
          type: "rollback",
          description: `Rollback to version ${versionNumber}`
        },
        createdBy: session.user,
        createdAt: new Date()
      }, { session });

      // 4. Restaurer avec nouveau numéro de version
      await db.collection('contents').updateOne(
        { _id: ObjectId(contentId) },
        {
          $set: {
            ...version.snapshot,
            version: newVersion,
            updatedAt: new Date()
          }
        },
        { session }
      );

      // 5. Créer entrée de version pour la restauration
      await db.collection('content_versions').insertOne({
        contentId: ObjectId(contentId),
        version: newVersion,
        snapshot: version.snapshot,
        changes: {
          type: "restore",
          description: `Restored from version ${versionNumber}`
        },
        createdBy: session.user,
        createdAt: new Date()
      }, { session });
    });

    return { success: true, newVersion };
  } finally {
    await session.endSession();
  }
}
```

### 4. Multi-langue (Internationalisation)

#### Option A : Documents séparés avec références

```javascript
// Collection: contents
// Version française
{
  _id: ObjectId("content_fr"),
  locale: "fr",
  slug: "introduction-mongodb",

  // Références aux traductions
  translations: [
    { locale: "en", contentId: ObjectId("content_en") },
    { locale: "es", contentId: ObjectId("content_es") },
    { locale: "de", contentId: ObjectId("content_de") }
  ],

  // Identifiant commun pour grouper les traductions
  translationGroup: "article_introduction_mongodb",

  data: {
    title: "Introduction à MongoDB",
    body: { /* contenu en français */ }
  }
}

// Version anglaise
{
  _id: ObjectId("content_en"),
  locale: "en",
  slug: "introduction-mongodb",

  translations: [
    { locale: "fr", contentId: ObjectId("content_fr") },
    { locale: "es", contentId: ObjectId("content_es") },
    { locale: "de", contentId: ObjectId("content_de") }
  ],

  translationGroup: "article_introduction_mongodb",

  data: {
    title: "Introduction to MongoDB",
    body: { /* content in English */ }
  }
}

// Index
db.contents.createIndex({ translationGroup: 1, locale: 1 });
db.contents.createIndex({ slug: 1, locale: 1 }, { unique: true });
```

**Avantages :**
- ✅ Isolation complète des langues
- ✅ Pas de limite de taille (chaque document indépendant)
- ✅ Performances optimales (pas de filtrage)
- ✅ Facilite le workflow de traduction

**Inconvénients :**
- ❌ Duplication de certaines métadonnées
- ❌ Complexité pour synchroniser les statuts

#### Option B : Document unique avec champs localisés

```javascript
// Collection: contents
{
  _id: ObjectId("..."),
  translationGroup: "article_introduction_mongodb",

  // Métadonnées communes (non localisées)
  type: "article",
  status: "published",
  author: { /* ... */ },
  categories: [ /* ... */ ],
  publishedAt: ISODate("..."),

  // Contenu localisé
  localizations: {
    fr: {
      slug: "introduction-mongodb",
      data: {
        title: "Introduction à MongoDB",
        excerpt: "MongoDB est une base...",
        body: { /* contenu français */ }
      },
      seo: {
        title: "Introduction à MongoDB | Guide complet",
        description: "Guide complet..."
      }
    },
    en: {
      slug: "introduction-mongodb",
      data: {
        title: "Introduction to MongoDB",
        excerpt: "MongoDB is a database...",
        body: { /* English content */ }
      },
      seo: {
        title: "Introduction to MongoDB | Complete Guide",
        description: "Complete guide..."
      }
    },
    es: {
      slug: "introduccion-mongodb",
      data: {
        title: "Introducción a MongoDB",
        excerpt: "MongoDB es una base...",
        body: { /* contenido español */ }
      },
      seo: {
        title: "Introducción a MongoDB | Guía completa",
        description: "Guía completa..."
      }
    }
  },

  // Langues disponibles
  availableLocales: ["fr", "en", "es"],
  defaultLocale: "fr"
}

// Index
db.contents.createIndex({ "localizations.fr.slug": 1 });
db.contents.createIndex({ "localizations.en.slug": 1 });
db.contents.createIndex({ "localizations.es.slug": 1 });
```

**Avantages :**
- ✅ Un seul document = cohérence garantie
- ✅ Métadonnées communes non dupliquées
- ✅ Requêtes simplifiées

**Inconvénients :**
- ❌ Risque d'atteindre limite 16 Mo avec beaucoup de langues
- ❌ Projection nécessaire pour chaque langue
- ❌ Index plus complexes

#### Recommandation : Hybride selon le contenu

```javascript
// Pour contenus courts (< 100 KB par langue) : Option B
// Pour contenus longs : Option A

class LocalizationStrategy {
  static determineStrategy(contentType, estimatedSize) {
    const SHORT_CONTENT_THRESHOLD = 100 * 1024;  // 100 KB

    if (estimatedSize < SHORT_CONTENT_THRESHOLD) {
      return 'embedded';  // Option B
    }

    return 'referenced';  // Option A
  }
}
```

### 5. Workflow éditorial et états

#### Modèle de workflow

```javascript
// Collection: contents
{
  _id: ObjectId("..."),

  // État du workflow
  workflow: {
    status: "in_review",  // draft, in_review, scheduled, published, archived

    // Historique des transitions
    history: [
      {
        from: "draft",
        to: "in_review",
        timestamp: ISODate("2024-12-08T10:00:00Z"),
        user: {
          id: ObjectId("..."),
          name: "John Writer",
          role: "author"
        },
        comment: "Ready for review"
      },
      {
        from: "in_review",
        to: "draft",
        timestamp: ISODate("2024-12-08T14:00:00Z"),
        user: {
          id: ObjectId("..."),
          name: "Jane Editor",
          role: "editor"
        },
        comment: "Please revise the introduction"
      },
      {
        from: "draft",
        to: "in_review",
        timestamp: ISODate("2024-12-09T09:00:00Z"),
        user: {
          id: ObjectId("..."),
          name: "John Writer",
          role: "author"
        },
        comment: "Introduction revised"
      }
    ],

    // Assignation
    assignedTo: {
      id: ObjectId("..."),
      name: "Jane Editor",
      role: "editor"
    },

    // Deadline
    dueDate: ISODate("2024-12-10T00:00:00Z")
  },

  // Publication programmée
  scheduling: {
    scheduledAt: ISODate("2024-12-15T08:00:00Z"),
    timezone: "Europe/Paris",
    autoPublish: true
  }
}

// Index pour workflow
db.contents.createIndex({
  "workflow.status": 1,
  "workflow.assignedTo.id": 1
});

db.contents.createIndex({
  "workflow.status": 1,
  "workflow.dueDate": 1
});

db.contents.createIndex({
  "scheduling.scheduledAt": 1,
  "workflow.status": 1
});
```

#### Service de gestion du workflow

```javascript
class WorkflowService {
  static TRANSITIONS = {
    draft: ['in_review', 'archived'],
    in_review: ['draft', 'scheduled', 'published', 'archived'],
    scheduled: ['draft', 'published', 'archived'],
    published: ['draft', 'archived'],
    archived: ['draft']
  };

  async transitionStatus(contentId, newStatus, userId, comment) {
    const content = await db.collection('contents')
      .findOne({ _id: ObjectId(contentId) });

    const currentStatus = content.workflow.status;

    // Valider la transition
    if (!this.canTransition(currentStatus, newStatus)) {
      throw new Error(
        `Cannot transition from ${currentStatus} to ${newStatus}`
      );
    }

    // Vérifier permissions
    await this.checkPermissions(userId, currentStatus, newStatus);

    // Appliquer la transition
    const user = await db.collection('users')
      .findOne({ _id: ObjectId(userId) });

    const transition = {
      from: currentStatus,
      to: newStatus,
      timestamp: new Date(),
      user: {
        id: user._id,
        name: user.profile.firstName + ' ' + user.profile.lastName,
        role: user.role
      },
      comment
    };

    await db.collection('contents').updateOne(
      { _id: ObjectId(contentId) },
      {
        $set: {
          'workflow.status': newStatus,
          updatedAt: new Date()
        },
        $push: {
          'workflow.history': transition
        }
      }
    );

    // Actions post-transition
    await this.postTransitionActions(contentId, newStatus);

    return { success: true, transition };
  }

  canTransition(from, to) {
    return this.TRANSITIONS[from]?.includes(to) || false;
  }

  async checkPermissions(userId, fromStatus, toStatus) {
    const user = await db.collection('users')
      .findOne({ _id: ObjectId(userId) });

    const requiredPermissions = {
      draft: ['author', 'editor', 'admin'],
      in_review: ['editor', 'admin'],
      scheduled: ['editor', 'admin'],
      published: ['editor', 'admin'],
      archived: ['editor', 'admin']
    };

    const allowedRoles = requiredPermissions[toStatus];

    if (!allowedRoles.includes(user.role)) {
      throw new Error('Insufficient permissions');
    }
  }

  async postTransitionActions(contentId, newStatus) {
    switch (newStatus) {
      case 'published':
        // Invalider cache
        await this.invalidateCache(contentId);

        // Notifier abonnés
        await this.notifySubscribers(contentId);

        // Déclencher webhook
        await this.triggerWebhook('content.published', contentId);
        break;

      case 'in_review':
        // Notifier éditeurs
        await this.notifyEditors(contentId);
        break;
    }
  }
}
```

### 6. Permissions et rôles éditoriaux

#### Modèle RBAC (Role-Based Access Control)

```javascript
// Collection: users
{
  _id: ObjectId("..."),
  email: "editor@example.com",

  // Rôle global
  role: "editor",  // viewer, author, editor, admin

  // Permissions granulaires
  permissions: [
    "content.create",
    "content.read",
    "content.update",
    "content.delete",
    "content.publish",
    "media.upload",
    "users.read"
  ],

  // Contraintes sur les contenus
  contentAccess: {
    // Peut modifier seulement certaines catégories
    categories: [ObjectId("cat_1"), ObjectId("cat_2")],

    // Peut modifier seulement ses propres contenus
    ownOnly: false,

    // Peut modifier les brouillons des autres
    canEditDrafts: true
  }
}

// Collection: contents (avec ACL)
{
  _id: ObjectId("..."),

  // Access Control List
  acl: {
    owner: ObjectId("author_id"),

    // Permissions spécifiques
    permissions: [
      {
        userId: ObjectId("editor_1"),
        actions: ["read", "update", "comment"]
      },
      {
        userId: ObjectId("editor_2"),
        actions: ["read", "comment"]
      }
    ],

    // Permissions par rôle
    roles: {
      author: ["read", "update"],
      editor: ["read", "update", "publish"],
      admin: ["read", "update", "publish", "delete"]
    }
  }
}
```

#### Service d'autorisation

```javascript
class AuthorizationService {
  async canPerformAction(userId, contentId, action) {
    const user = await db.collection('users')
      .findOne({ _id: ObjectId(userId) });

    const content = await db.collection('contents')
      .findOne({ _id: ObjectId(contentId) });

    // Admin peut tout faire
    if (user.role === 'admin') {
      return true;
    }

    // Vérifier si propriétaire
    if (content.acl.owner.equals(user._id)) {
      return true;
    }

    // Vérifier permissions utilisateur spécifiques
    const userPermission = content.acl.permissions.find(
      p => p.userId.equals(user._id)
    );

    if (userPermission?.actions.includes(action)) {
      return true;
    }

    // Vérifier permissions par rôle
    const rolePermissions = content.acl.roles[user.role];
    if (rolePermissions?.includes(action)) {
      return true;
    }

    // Vérifier contraintes de contenu
    if (content.workflow.status === 'draft' &&
        user.contentAccess.canEditDrafts) {
      return ['read', 'update'].includes(action);
    }

    return false;
  }

  async filterContentsByPermission(userId, query, action = 'read') {
    const user = await db.collection('users')
      .findOne({ _id: ObjectId(userId) });

    // Admin voit tout
    if (user.role === 'admin') {
      return query;
    }

    // Construire filtre basé sur les permissions
    const permissionFilter = {
      $or: [
        // Propriétaire
        { 'acl.owner': user._id },

        // Permission utilisateur explicite
        {
          'acl.permissions': {
            $elemMatch: {
              userId: user._id,
              actions: action
            }
          }
        },

        // Permission par rôle
        { [`acl.roles.${user.role}`]: action }
      ]
    };

    // Ajouter contraintes de catégories si applicable
    if (user.contentAccess.categories?.length > 0) {
      permissionFilter['categories.id'] = {
        $in: user.contentAccess.categories
      };
    }

    return { ...query, ...permissionFilter };
  }
}
```

### 7. Médias et assets

#### Modèle pour bibliothèque de médias

```javascript
// Collection: media
{
  _id: ObjectId("..."),

  // Informations du fichier
  filename: "sunset-beach-2024.jpg",
  originalFilename: "DSC_1234.jpg",

  // URLs (après upload vers CDN/S3)
  urls: {
    original: "https://cdn.example.com/media/sunset-beach-2024.jpg",
    large: "https://cdn.example.com/media/sunset-beach-2024-lg.jpg",
    medium: "https://cdn.example.com/media/sunset-beach-2024-md.jpg",
    small: "https://cdn.example.com/media/sunset-beach-2024-sm.jpg",
    thumbnail: "https://cdn.example.com/media/sunset-beach-2024-thumb.jpg"
  },

  // Type et métadonnées
  type: "image",  // image, video, audio, document
  mimeType: "image/jpeg",
  size: 2457893,  // bytes

  // Métadonnées d'image
  metadata: {
    width: 4000,
    height: 3000,
    aspectRatio: 1.33,
    format: "jpg",
    colorSpace: "sRGB",

    // EXIF data
    exif: {
      camera: "Nikon D850",
      lens: "24-70mm f/2.8",
      iso: 100,
      aperture: 8,
      shutterSpeed: "1/250",
      focalLength: 35,
      dateTaken: ISODate("2024-06-15T18:30:00Z"),
      gps: {
        latitude: 34.0095,
        longitude: -118.4912
      }
    }
  },

  // Informations éditoriales
  title: "Beautiful Sunset at Santa Monica Beach",
  alt: "Sunset over Santa Monica pier with orange sky",
  caption: "Captured during golden hour",
  description: "A stunning sunset view from Santa Monica beach...",

  // Taxonomie
  tags: ["sunset", "beach", "california", "nature"],
  categories: [ObjectId("nature"), ObjectId("travel")],

  // Propriétaire et permissions
  uploadedBy: {
    id: ObjectId("..."),
    name: "John Photographer"
  },

  // Dates
  uploadedAt: ISODate("2024-12-09T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T10:00:00Z"),

  // Utilisation
  usageCount: 5,  // Nombre de contenus utilisant ce média
  usedIn: [
    ObjectId("content_1"),
    ObjectId("content_2"),
    ObjectId("content_3")
  ],

  // SEO et optimisation
  seo: {
    keywords: ["sunset", "beach", "california"],
    alt: "Beautiful sunset at Santa Monica Beach"
  }
}

// Index pour médias
db.media.createIndex({ type: 1, uploadedAt: -1 });
db.media.createIndex({ tags: 1 });
db.media.createIndex({ "uploadedBy.id": 1, uploadedAt: -1 });
db.media.createIndex({ filename: 1 });
db.media.createIndex(
  { title: "text", alt: "text", tags: "text" },
  { weights: { title: 10, alt: 5, tags: 3 } }
);
```

#### Service de gestion des médias

```javascript
class MediaService {
  async uploadMedia(file, metadata, userId) {
    // 1. Upload vers S3/CDN
    const uploadedUrls = await this.uploadToStorage(file);

    // 2. Générer thumbnails et variations
    const variations = await this.generateVariations(file);

    // 3. Extraire métadonnées
    const exifData = await this.extractExif(file);

    // 4. Créer document média
    const mediaDoc = {
      filename: this.generateUniqueFilename(file),
      originalFilename: file.originalname,
      urls: { ...uploadedUrls, ...variations },
      type: this.getMediaType(file.mimetype),
      mimeType: file.mimetype,
      size: file.size,
      metadata: {
        width: exifData.width,
        height: exifData.height,
        aspectRatio: exifData.width / exifData.height,
        format: file.format,
        exif: exifData
      },
      title: metadata.title || file.originalname,
      alt: metadata.alt || '',
      tags: metadata.tags || [],
      uploadedBy: {
        id: ObjectId(userId),
        name: metadata.uploaderName
      },
      uploadedAt: new Date(),
      usageCount: 0,
      usedIn: []
    };

    const result = await db.collection('media').insertOne(mediaDoc);

    return { success: true, mediaId: result.insertedId, urls: uploadedUrls };
  }

  async trackMediaUsage(mediaId, contentId) {
    await db.collection('media').updateOne(
      { _id: ObjectId(mediaId) },
      {
        $inc: { usageCount: 1 },
        $addToSet: { usedIn: ObjectId(contentId) }
      }
    );
  }

  async deleteMedia(mediaId) {
    const media = await db.collection('media')
      .findOne({ _id: ObjectId(mediaId) });

    // Vérifier si utilisé
    if (media.usageCount > 0) {
      throw new Error(
        `Cannot delete media: used in ${media.usageCount} content(s)`
      );
    }

    // Supprimer du storage
    await this.deleteFromStorage(media.urls);

    // Supprimer du DB
    await db.collection('media').deleteOne({ _id: ObjectId(mediaId) });

    return { success: true };
  }
}
```

### 8. Performance et cache

#### Stratégie de cache multi-niveaux

```javascript
// Niveau 1: CDN Edge Cache (Cloudflare, Fastly)
// - Cache les pages publiques
// - TTL: 1 heure par défaut
// - Purge sur publication

// Niveau 2: Application Cache (Redis)
class ContentCacheService {
  constructor(redis, db) {
    this.redis = redis;
    this.db = db;
    this.TTL = {
      content: 3600,        // 1 heure
      list: 300,            // 5 minutes
      menu: 86400,          // 24 heures
      category: 3600        // 1 heure
    };
  }

  async getContent(slug, locale = 'fr') {
    const cacheKey = `content:${locale}:${slug}`;

    // Essayer cache
    let content = await this.redis.get(cacheKey);

    if (content) {
      return JSON.parse(content);
    }

    // Récupérer de MongoDB
    content = await this.db.collection('contents').findOne({
      slug,
      locale,
      status: 'published'
    });

    if (content) {
      // Stocker en cache
      await this.redis.setex(
        cacheKey,
        this.TTL.content,
        JSON.stringify(content)
      );
    }

    return content;
  }

  async invalidateContent(contentId) {
    const content = await this.db.collection('contents')
      .findOne({ _id: ObjectId(contentId) });

    // Invalider toutes les localisations
    const promises = [];

    if (content.translations) {
      for (const trans of content.translations) {
        const key = `content:${trans.locale}:${content.slug}`;
        promises.push(this.redis.del(key));
      }
    }

    // Invalider aussi les listes
    promises.push(this.redis.del('content:list:*'));

    await Promise.all(promises);

    // Purger CDN
    await this.purgeCDN(content.slug);
  }

  async purgeCDN(slug) {
    // Exemple avec Cloudflare
    await fetch('https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.CLOUDFLARE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        files: [
          `https://example.com/${slug}`,
          `https://example.com/api/content/${slug}`
        ]
      })
    });
  }
}
```

#### Optimisation des requêtes de listing

```javascript
// Pattern: Pagination efficace avec agrégation
async function getContentList(options = {}) {
  const {
    category,
    tags,
    status = 'published',
    locale = 'fr',
    page = 1,
    limit = 20,
    sortBy = 'publishedAt',
    sortOrder = -1
  } = options;

  // Construire pipeline d'agrégation
  const pipeline = [];

  // 1. Match principal
  const matchStage = {
    status,
    locale
  };

  if (category) {
    matchStage['categories.id'] = ObjectId(category);
  }

  if (tags?.length > 0) {
    matchStage.tags = { $in: tags };
  }

  pipeline.push({ $match: matchStage });

  // 2. Projection (champs essentiels seulement)
  pipeline.push({
    $project: {
      slug: 1,
      type: 1,
      'data.title': 1,
      'data.excerpt': 1,
      'data.featuredImage': 1,
      author: 1,
      publishedAt: 1,
      'stats.views': 1,
      'stats.likes': 1,
      categories: 1,
      tags: 1
    }
  });

  // 3. Tri
  pipeline.push({
    $sort: { [sortBy]: sortOrder }
  });

  // 4. Pagination avec $facet pour count + données
  pipeline.push({
    $facet: {
      metadata: [
        { $count: 'total' },
        {
          $addFields: {
            page,
            limit,
            pages: { $ceil: { $divide: ['$total', limit] } }
          }
        }
      ],
      data: [
        { $skip: (page - 1) * limit },
        { $limit: limit }
      ]
    }
  });

  const result = await db.collection('contents')
    .aggregate(pipeline)
    .toArray();

  return {
    metadata: result[0].metadata[0] || { total: 0, page, limit, pages: 0 },
    data: result[0].data
  };
}
```

### 9. Recherche et indexation

#### Option A : MongoDB Text Search (simple)

```javascript
// Index text déjà créé
db.contents.createIndex({
  searchText: "text",
  "data.title": "text",
  tags: "text"
});

// Recherche simple
async function searchContent(query, locale = 'fr') {
  return db.collection('contents').find({
    $text: { $search: query },
    locale,
    status: 'published'
  }, {
    score: { $meta: 'textScore' }
  })
  .sort({ score: { $meta: 'textScore' } })
  .limit(20)
  .toArray();
}
```

#### Option B : Elasticsearch (avancé)

```javascript
// Service d'indexation Elasticsearch
class SearchService {
  constructor(esClient, mongoDb) {
    this.es = esClient;
    this.db = mongoDb;
  }

  async indexContent(contentId) {
    const content = await this.db.collection('contents')
      .findOne({ _id: ObjectId(contentId) });

    // Mapper vers index Elasticsearch
    const doc = {
      id: content._id.toString(),
      slug: content.slug,
      type: content.type,
      title: content.data.title,
      excerpt: content.data.excerpt,
      body: this.extractTextFromBlocks(content.data.body.blocks),
      author: content.author.name,
      categories: content.categories.map(c => c.name),
      tags: content.tags,
      publishedAt: content.publishedAt,
      locale: content.locale,
      status: content.status
    };

    await this.es.index({
      index: 'contents',
      id: content._id.toString(),
      body: doc
    });
  }

  async search(query, options = {}) {
    const {
      locale = 'fr',
      categories = [],
      tags = [],
      dateFrom,
      dateTo,
      page = 1,
      size = 20
    } = options;

    // Construire requête Elasticsearch
    const mustClauses = [
      {
        multi_match: {
          query,
          fields: ['title^3', 'excerpt^2', 'body', 'tags^2'],
          type: 'best_fields',
          fuzziness: 'AUTO'
        }
      },
      { term: { locale } },
      { term: { status: 'published' } }
    ];

    if (categories.length > 0) {
      mustClauses.push({
        terms: { 'categories.keyword': categories }
      });
    }

    if (tags.length > 0) {
      mustClauses.push({
        terms: { 'tags': tags }
      });
    }

    if (dateFrom || dateTo) {
      const rangeClause = { range: { publishedAt: {} } };
      if (dateFrom) rangeClause.range.publishedAt.gte = dateFrom;
      if (dateTo) rangeClause.range.publishedAt.lte = dateTo;
      mustClauses.push(rangeClause);
    }

    const result = await this.es.search({
      index: 'contents',
      body: {
        from: (page - 1) * size,
        size,
        query: {
          bool: {
            must: mustClauses
          }
        },
        highlight: {
          fields: {
            title: {},
            excerpt: {},
            body: {
              fragment_size: 150,
              number_of_fragments: 3
            }
          }
        }
      }
    });

    return {
      total: result.hits.total.value,
      hits: result.hits.hits.map(hit => ({
        ...hit._source,
        score: hit._score,
        highlights: hit.highlight
      }))
    };
  }

  extractTextFromBlocks(blocks) {
    return blocks
      .filter(b => ['paragraph', 'heading'].includes(b.type))
      .map(b => b.data.text)
      .join(' ');
  }
}
```

#### Synchronisation MongoDB → Elasticsearch avec Change Streams

```javascript
async function watchContentChanges() {
  const changeStream = db.collection('contents').watch([
    {
      $match: {
        'operationType': { $in: ['insert', 'update', 'replace', 'delete'] }
      }
    }
  ]);

  const searchService = new SearchService(esClient, db);

  changeStream.on('change', async (change) => {
    try {
      switch (change.operationType) {
        case 'insert':
        case 'update':
        case 'replace':
          await searchService.indexContent(change.documentKey._id);
          console.log(`Indexed content: ${change.documentKey._id}`);
          break;

        case 'delete':
          await esClient.delete({
            index: 'contents',
            id: change.documentKey._id.toString()
          });
          console.log(`Deleted from index: ${change.documentKey._id}`);
          break;
      }
    } catch (error) {
      console.error('Error syncing to Elasticsearch:', error);
    }
  });

  console.log('Watching content changes...');
}
```

## Déploiement et architecture de production

### Configuration MongoDB pour CMS

```yaml
# Configuration Replica Set recommandée
Replica Set: 3 nœuds minimum
  - Primary: Toutes les écritures
  - Secondary 1: Lectures (admin dashboard, analytics)
  - Secondary 2: Backup et analytics lourds

# Write Concern
writeConcern:
  w: "majority"    # Pour contenu critique
  j: true          # Journaling activé
  wtimeout: 5000   # Timeout 5 secondes

# Read Preference
readPreference:
  - Admin Dashboard: "primaryPreferred"  # Données fraîches
  - Frontend Public: "secondary"         # Performance
  - Analytics: "secondary"               # Décharger primary

# Index Configuration
indexes:
  - Tous les champs de filtrage (status, locale, categories)
  - Text search pour recherche basique
  - Géospatial si contenu localisé
  - TTL pour contenu temporaire (événements)
```

### Checklist de déploiement CMS

#### ✅ Modélisation et schéma

- [ ] Schéma validé avec JSON Schema
- [ ] Versioning configuré
- [ ] Multi-langue implémenté
- [ ] Workflow éditorial défini
- [ ] Permissions et ACL configurées

#### ✅ Performance

- [ ] Index optimisés pour toutes requêtes fréquentes
- [ ] Cache multi-niveaux (CDN + Redis)
- [ ] Pagination cursor-based
- [ ] Images optimisées et CDN configuré
- [ ] Lazy loading pour médias

#### ✅ Recherche

- [ ] Elasticsearch déployé (si nécessaire)
- [ ] Change Streams pour synchronisation
- [ ] Fallback sur MongoDB text search
- [ ] Auto-complétion configurée
- [ ] Filtres facettés implémentés

#### ✅ Sécurité

- [ ] Authentification robuste (JWT + refresh tokens)
- [ ] RBAC implémenté
- [ ] Content ACL configuré
- [ ] Rate limiting sur API
- [ ] CORS configuré correctement
- [ ] Validation des uploads (taille, type)

#### ✅ Opérations

- [ ] Backup automatique quotidien
- [ ] Monitoring des performances
- [ ] Logs centralisés
- [ ] Alertes configurées
- [ ] Plan de disaster recovery
- [ ] Documentation à jour

## Conclusion

MongoDB est particulièrement adapté aux CMS modernes grâce à :

**✅ Forces démontrées :**
- Flexibilité de schéma pour types de contenu variés
- Documents imbriqués pour hiérarchies complexes
- Agrégation puissante pour taxonomies et filtres
- Change Streams pour synchronisation temps réel
- Versioning natif avec collections séparées
- Performance en lecture pour frontend public

**⚠️ Considérations importantes :**
- Dénormalisation nécessite synchronisation rigoureuse
- Index multiples impactent les écritures
- Elasticsearch recommandé pour recherche avancée
- CDN essentiel pour performance globale
- Workflow de cache cohérent crucial

**🎯 Patterns essentiels pour CMS :**
1. **Attribute Pattern** pour contenu polymorphe
2. **Extended Reference** pour performances
3. **Materialized Path** pour hiérarchies
4. **Versioning Pattern** pour historique
5. **Computed Pattern** pour statistiques

Cette architecture supporte des CMS allant de petits blogs à des plateformes de contenu à grande échelle avec des millions de pages et d'utilisateurs.

---

**Références :**
- MongoDB Blog: "Building a CMS with MongoDB"
- Contentful Architecture Blog
- Strapi Technical Documentation
- "Web Content Management" - Deane Barker

⏭️ [Catalogue produits (e-commerce)](/20-cas-usage-architectures/03-catalogue-produits-ecommerce.md)
