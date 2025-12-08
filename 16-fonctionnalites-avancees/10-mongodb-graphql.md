🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.10 MongoDB et GraphQL

## Introduction

**GraphQL** est un langage de requête pour API développé par Facebook qui permet aux clients de demander exactement les données dont ils ont besoin. Contrairement aux API REST traditionnelles, GraphQL offre :

- **Requêtes flexibles** : Le client spécifie précisément les champs souhaités
- **Un seul endpoint** : Plus besoin de multiples endpoints REST
- **Typage fort** : Schéma explicite avec validation automatique
- **Résolution de problème N+1** : Avec DataLoader et batching
- **Subscriptions temps réel** : Mises à jour en temps réel via WebSocket

L'intégration MongoDB + GraphQL permet de créer des APIs modernes, performantes et flexibles.

---

## Configuration de base

### Installation des dépendances

```javascript
// package.json
{
  "dependencies": {
    "@apollo/server": "^4.10.0",
    "graphql": "^16.8.1",
    "mongodb": "^6.3.0",
    "dataloader": "^2.2.2",
    "graphql-scalars": "^1.22.4",
    "graphql-subscriptions": "^2.0.0",
    "ws": "^8.16.0"
  }
}
```

### Serveur Apollo Server avec MongoDB

```javascript
const { ApolloServer } = require('@apollo/server');
const { startStandaloneServer } = require('@apollo/server/standalone');
const { MongoClient, ObjectId } = require('mongodb');

// Type definitions (schéma GraphQL)
const typeDefs = `#graphql
  scalar DateTime
  scalar ObjectID

  type User {
    id: ObjectID!
    name: String!
    email: String!
    posts: [Post!]!
    createdAt: DateTime!
  }

  type Post {
    id: ObjectID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
    published: Boolean!
    createdAt: DateTime!
  }

  type Comment {
    id: ObjectID!
    content: String!
    author: User!
    post: Post!
    createdAt: DateTime!
  }

  type Query {
    users: [User!]!
    user(id: ObjectID!): User
    posts(published: Boolean): [Post!]!
    post(id: ObjectID!): Post
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
    createPost(
      title: String!
      content: String!
      authorId: ObjectID!
    ): Post!
    createComment(
      content: String!
      postId: ObjectID!
      authorId: ObjectID!
    ): Comment!
  }
`;

// Resolvers
const resolvers = {
  // Custom scalars
  ObjectID: {
    serialize: (value) => value.toString(),
    parseValue: (value) => new ObjectId(value),
    parseLiteral: (ast) => new ObjectId(ast.value)
  },

  DateTime: {
    serialize: (value) => value.toISOString(),
    parseValue: (value) => new Date(value),
    parseLiteral: (ast) => new Date(ast.value)
  },

  // Type resolvers
  User: {
    id: (parent) => parent._id,
    posts: async (parent, args, context) => {
      return await context.db.collection('posts')
        .find({ authorId: parent._id })
        .toArray();
    }
  },

  Post: {
    id: (parent) => parent._id,
    author: async (parent, args, context) => {
      return await context.db.collection('users')
        .findOne({ _id: parent.authorId });
    },
    comments: async (parent, args, context) => {
      return await context.db.collection('comments')
        .find({ postId: parent._id })
        .toArray();
    }
  },

  Comment: {
    id: (parent) => parent._id,
    author: async (parent, args, context) => {
      return await context.db.collection('users')
        .findOne({ _id: parent.authorId });
    },
    post: async (parent, args, context) => {
      return await context.db.collection('posts')
        .findOne({ _id: parent.postId });
    }
  },

  // Query resolvers
  Query: {
    users: async (parent, args, context) => {
      return await context.db.collection('users')
        .find()
        .toArray();
    },

    user: async (parent, { id }, context) => {
      return await context.db.collection('users')
        .findOne({ _id: id });
    },

    posts: async (parent, { published }, context) => {
      const filter = published !== undefined
        ? { published }
        : {};

      return await context.db.collection('posts')
        .find(filter)
        .toArray();
    },

    post: async (parent, { id }, context) => {
      return await context.db.collection('posts')
        .findOne({ _id: id });
    }
  },

  // Mutation resolvers
  Mutation: {
    createUser: async (parent, { name, email }, context) => {
      const user = {
        name,
        email,
        createdAt: new Date()
      };

      const result = await context.db.collection('users')
        .insertOne(user);

      return { ...user, _id: result.insertedId };
    },

    createPost: async (parent, { title, content, authorId }, context) => {
      const post = {
        title,
        content,
        authorId,
        published: false,
        createdAt: new Date()
      };

      const result = await context.db.collection('posts')
        .insertOne(post);

      return { ...post, _id: result.insertedId };
    },

    createComment: async (parent, { content, postId, authorId }, context) => {
      const comment = {
        content,
        postId,
        authorId,
        createdAt: new Date()
      };

      const result = await context.db.collection('comments')
        .insertOne(comment);

      return { ...comment, _id: result.insertedId };
    }
  }
};

// Démarrer le serveur
async function startServer() {
  const client = new MongoClient('mongodb://localhost:27017');
  await client.connect();
  const db = client.db('graphql_demo');

  const server = new ApolloServer({
    typeDefs,
    resolvers
  });

  const { url } = await startStandaloneServer(server, {
    context: async () => ({ db }),
    listen: { port: 4000 }
  });

  console.log(`🚀 Server ready at ${url}`);
}

startServer();
```

### Exemples de requêtes GraphQL

```graphql
# Créer un utilisateur
mutation {
  createUser(name: "Alice", email: "alice@example.com") {
    id
    name
    email
    createdAt
  }
}

# Récupérer utilisateur avec ses posts
query {
  user(id: "507f1f77bcf86cd799439011") {
    id
    name
    email
    posts {
      id
      title
      published
      comments {
        id
        content
        author {
          name
        }
      }
    }
  }
}

# Requête flexible - seulement les champs nécessaires
query {
  posts(published: true) {
    title
    author {
      name
    }
  }
}
```

---

## Optimisation avec DataLoader

### Problème N+1

```javascript
// ❌ PROBLÈME: N+1 queries
// Si on récupère 100 posts, et que chaque post charge son author,
// on fait 1 requête pour les posts + 100 requêtes pour les authors

const Post = {
  author: async (parent, args, context) => {
    // Cette requête est exécutée pour CHAQUE post
    return await context.db.collection('users')
      .findOne({ _id: parent.authorId });
  }
};

// Résultat: 101 requêtes au lieu de 2 !
```

### Solution avec DataLoader

```javascript
const DataLoader = require('dataloader');

class DataLoaders {
  constructor(db) {
    this.db = db;
  }

  // Loader pour utilisateurs
  userLoader() {
    return new DataLoader(async (userIds) => {
      // Batch: Une seule requête pour tous les IDs
      const users = await this.db.collection('users')
        .find({ _id: { $in: userIds } })
        .toArray();

      // Mapper résultats dans le bon ordre
      const userMap = new Map(
        users.map(user => [user._id.toString(), user])
      );

      return userIds.map(id => userMap.get(id.toString()) || null);
    });
  }

  // Loader pour posts par auteur
  postsByAuthorLoader() {
    return new DataLoader(async (authorIds) => {
      const posts = await this.db.collection('posts')
        .find({ authorId: { $in: authorIds } })
        .toArray();

      // Grouper posts par authorId
      const postsByAuthor = new Map();

      authorIds.forEach(id => {
        postsByAuthor.set(id.toString(), []);
      });

      posts.forEach(post => {
        const key = post.authorId.toString();
        if (postsByAuthor.has(key)) {
          postsByAuthor.get(key).push(post);
        }
      });

      return authorIds.map(id => postsByAuthor.get(id.toString()) || []);
    });
  }

  // Loader pour commentaires par post
  commentsByPostLoader() {
    return new DataLoader(async (postIds) => {
      const comments = await this.db.collection('comments')
        .find({ postId: { $in: postIds } })
        .toArray();

      const commentsByPost = new Map();

      postIds.forEach(id => {
        commentsByPost.set(id.toString(), []);
      });

      comments.forEach(comment => {
        const key = comment.postId.toString();
        if (commentsByPost.has(key)) {
          commentsByPost.get(key).push(comment);
        }
      });

      return postIds.map(id => commentsByPost.get(id.toString()) || []);
    });
  }
}

// Mettre à jour le serveur pour utiliser DataLoader
const server = new ApolloServer({
  typeDefs,
  resolvers
});

await startStandaloneServer(server, {
  context: async () => {
    const loaders = new DataLoaders(db);

    return {
      db,
      loaders: {
        user: loaders.userLoader(),
        postsByAuthor: loaders.postsByAuthorLoader(),
        commentsByPost: loaders.commentsByPostLoader()
      }
    };
  }
});

// Mettre à jour les resolvers
const resolvers = {
  User: {
    posts: async (parent, args, context) => {
      // Utilise DataLoader au lieu de requête directe
      return await context.loaders.postsByAuthor.load(parent._id);
    }
  },

  Post: {
    author: async (parent, args, context) => {
      // Batch automatique des requêtes d'authors
      return await context.loaders.user.load(parent.authorId);
    },
    comments: async (parent, args, context) => {
      return await context.loaders.commentsByPost.load(parent._id);
    }
  },

  Comment: {
    author: async (parent, args, context) => {
      return await context.loaders.user.load(parent.authorId);
    }
  }
};

// ✅ RÉSULTAT:
// 100 posts -> 3 requêtes au lieu de 101+
// 1. Requête posts
// 2. Batch requête authors (tous les authorIds uniques)
// 3. Batch requête comments (tous les postIds)
```

---

## Cas d'usage avancés

### Cas 1 : E-commerce avec GraphQL

```javascript
const typeDefs = `#graphql
  scalar DateTime
  scalar ObjectID

  type Product {
    id: ObjectID!
    name: String!
    description: String!
    price: Float!
    stock: Int!
    category: Category!
    images: [String!]!
    reviews: [Review!]!
    averageRating: Float
    reviewCount: Int!
    variants: [ProductVariant!]!
    relatedProducts: [Product!]!
  }

  type ProductVariant {
    id: ObjectID!
    sku: String!
    name: String!
    price: Float!
    stock: Int!
    attributes: [VariantAttribute!]!
  }

  type VariantAttribute {
    name: String!
    value: String!
  }

  type Category {
    id: ObjectID!
    name: String!
    slug: String!
    parent: Category
    children: [Category!]!
    products(
      limit: Int = 20
      offset: Int = 0
      sortBy: ProductSortInput
    ): ProductConnection!
  }

  input ProductSortInput {
    field: ProductSortField!
    order: SortOrder!
  }

  enum ProductSortField {
    NAME
    PRICE
    CREATED_AT
    RATING
  }

  enum SortOrder {
    ASC
    DESC
  }

  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type ProductEdge {
    node: Product!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  type Review {
    id: ObjectID!
    rating: Int!
    title: String!
    content: String!
    author: User!
    product: Product!
    helpful: Int!
    createdAt: DateTime!
  }

  type User {
    id: ObjectID!
    name: String!
    email: String!
    orders: [Order!]!
    cart: Cart
    wishlist: [Product!]!
  }

  type Cart {
    id: ObjectID!
    items: [CartItem!]!
    total: Float!
  }

  type CartItem {
    product: Product!
    variant: ProductVariant
    quantity: Int!
    price: Float!
    subtotal: Float!
  }

  type Order {
    id: ObjectID!
    orderNumber: String!
    user: User!
    items: [OrderItem!]!
    status: OrderStatus!
    total: Float!
    shippingAddress: Address!
    createdAt: DateTime!
    updatedAt: DateTime!
  }

  type OrderItem {
    product: Product!
    variant: ProductVariant
    quantity: Int!
    price: Float!
    subtotal: Float!
  }

  enum OrderStatus {
    PENDING
    PROCESSING
    SHIPPED
    DELIVERED
    CANCELLED
  }

  type Address {
    street: String!
    city: String!
    state: String!
    postalCode: String!
    country: String!
  }

  type Query {
    # Produits
    product(id: ObjectID!): Product
    products(
      categoryId: ObjectID
      search: String
      minPrice: Float
      maxPrice: Float
      minRating: Float
      limit: Int = 20
      after: String
    ): ProductConnection!

    # Catégories
    category(id: ObjectID, slug: String): Category
    categories: [Category!]!

    # Utilisateur
    me: User
    cart: Cart

    # Recherche
    searchProducts(query: String!, limit: Int = 20): [Product!]!
  }

  type Mutation {
    # Panier
    addToCart(
      productId: ObjectID!
      variantId: ObjectID
      quantity: Int!
    ): Cart!

    updateCartItem(
      itemId: ObjectID!
      quantity: Int!
    ): Cart!

    removeFromCart(itemId: ObjectID!): Cart!
    clearCart: Cart!

    # Commande
    createOrder(
      shippingAddress: AddressInput!
      paymentMethod: String!
    ): Order!

    # Review
    createReview(
      productId: ObjectID!
      rating: Int!
      title: String!
      content: String!
    ): Review!

    # Wishlist
    addToWishlist(productId: ObjectID!): User!
    removeFromWishlist(productId: ObjectID!): User!
  }

  input AddressInput {
    street: String!
    city: String!
    state: String!
    postalCode: String!
    country: String!
  }

  type Subscription {
    orderUpdated(userId: ObjectID!): Order!
    stockChanged(productId: ObjectID!): Product!
  }
`;

// Resolvers complets
const resolvers = {
  Query: {
    product: async (parent, { id }, context) => {
      return await context.db.collection('products')
        .findOne({ _id: id });
    },

    products: async (parent, args, context) => {
      const {
        categoryId,
        search,
        minPrice,
        maxPrice,
        minRating,
        limit = 20,
        after
      } = args;

      // Construire filtre
      const filter = {};

      if (categoryId) {
        filter.categoryId = categoryId;
      }

      if (search) {
        filter.$text = { $search: search };
      }

      if (minPrice !== undefined || maxPrice !== undefined) {
        filter.price = {};
        if (minPrice) filter.price.$gte = minPrice;
        if (maxPrice) filter.price.$lte = maxPrice;
      }

      if (minRating) {
        filter.averageRating = { $gte: minRating };
      }

      // Cursor pagination
      if (after) {
        const cursor = Buffer.from(after, 'base64').toString('ascii');
        filter._id = { $gt: new ObjectId(cursor) };
      }

      // Exécuter requête
      const products = await context.db.collection('products')
        .find(filter)
        .limit(limit + 1)
        .toArray();

      // Construire réponse avec pagination
      const hasNextPage = products.length > limit;
      const nodes = hasNextPage ? products.slice(0, limit) : products;

      const edges = nodes.map(node => ({
        node,
        cursor: Buffer.from(node._id.toString()).toString('base64')
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: !!after,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor
        },
        totalCount: await context.db.collection('products')
          .countDocuments(filter)
      };
    },

    category: async (parent, { id, slug }, context) => {
      const filter = id ? { _id: id } : { slug };
      return await context.db.collection('categories')
        .findOne(filter);
    },

    categories: async (parent, args, context) => {
      return await context.db.collection('categories')
        .find({ parentId: null })
        .toArray();
    },

    me: async (parent, args, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      return await context.loaders.user.load(context.userId);
    },

    cart: async (parent, args, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      return await context.db.collection('carts')
        .findOne({ userId: context.userId });
    },

    searchProducts: async (parent, { query, limit }, context) => {
      // Recherche full-text
      const products = await context.db.collection('products')
        .aggregate([
          {
            $search: {
              index: "products_search",
              text: {
                query: query,
                path: ["name", "description"],
                fuzzy: { maxEdits: 2 }
              }
            }
          },
          { $limit: limit },
          {
            $addFields: {
              searchScore: { $meta: "searchScore" }
            }
          }
        ])
        .toArray();

      return products;
    }
  },

  Product: {
    id: (parent) => parent._id,

    category: async (parent, args, context) => {
      return await context.loaders.category.load(parent.categoryId);
    },

    reviews: async (parent, args, context) => {
      return await context.loaders.reviewsByProduct.load(parent._id);
    },

    averageRating: async (parent, args, context) => {
      const reviews = await context.loaders.reviewsByProduct.load(parent._id);

      if (reviews.length === 0) return null;

      const sum = reviews.reduce((acc, r) => acc + r.rating, 0);
      return sum / reviews.length;
    },

    reviewCount: async (parent, args, context) => {
      const reviews = await context.loaders.reviewsByProduct.load(parent._id);
      return reviews.length;
    },

    variants: async (parent, args, context) => {
      return await context.db.collection('product_variants')
        .find({ productId: parent._id })
        .toArray();
    },

    relatedProducts: async (parent, args, context) => {
      // Trouver produits similaires (même catégorie)
      return await context.db.collection('products')
        .find({
          categoryId: parent.categoryId,
          _id: { $ne: parent._id }
        })
        .limit(5)
        .toArray();
    }
  },

  Category: {
    id: (parent) => parent._id,

    parent: async (parent, args, context) => {
      if (!parent.parentId) return null;
      return await context.loaders.category.load(parent.parentId);
    },

    children: async (parent, args, context) => {
      return await context.db.collection('categories')
        .find({ parentId: parent._id })
        .toArray();
    },

    products: async (parent, args, context) => {
      const { limit = 20, offset = 0, sortBy } = args;

      // Construire sort
      let sort = {};
      if (sortBy) {
        const field = sortBy.field.toLowerCase();
        sort[field] = sortBy.order === 'ASC' ? 1 : -1;
      }

      const products = await context.db.collection('products')
        .find({ categoryId: parent._id })
        .sort(sort)
        .skip(offset)
        .limit(limit + 1)
        .toArray();

      const hasMore = products.length > limit;
      const nodes = hasMore ? products.slice(0, limit) : products;

      const edges = nodes.map((node, index) => ({
        node,
        cursor: Buffer.from((offset + index).toString()).toString('base64')
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage: hasMore,
          hasPreviousPage: offset > 0,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor
        },
        totalCount: await context.db.collection('products')
          .countDocuments({ categoryId: parent._id })
      };
    }
  },

  Cart: {
    id: (parent) => parent._id,

    items: async (parent, args, context) => {
      return parent.items || [];
    },

    total: (parent) => {
      return parent.items.reduce((sum, item) => sum + item.subtotal, 0);
    }
  },

  CartItem: {
    product: async (parent, args, context) => {
      return await context.loaders.product.load(parent.productId);
    },

    variant: async (parent, args, context) => {
      if (!parent.variantId) return null;
      return await context.db.collection('product_variants')
        .findOne({ _id: parent.variantId });
    },

    subtotal: (parent) => {
      return parent.price * parent.quantity;
    }
  },

  Mutation: {
    addToCart: async (parent, { productId, variantId, quantity }, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      // Récupérer produit
      const product = await context.db.collection('products')
        .findOne({ _id: productId });

      if (!product) {
        throw new Error('Product not found');
      }

      // Vérifier stock
      if (product.stock < quantity) {
        throw new Error('Insufficient stock');
      }

      // Prix (variant ou product)
      let price = product.price;
      if (variantId) {
        const variant = await context.db.collection('product_variants')
          .findOne({ _id: variantId });
        price = variant.price;
      }

      // Ajouter au panier
      const result = await context.db.collection('carts')
        .findOneAndUpdate(
          { userId: context.userId },
          {
            $push: {
              items: {
                _id: new ObjectId(),
                productId,
                variantId,
                quantity,
                price,
                subtotal: price * quantity,
                addedAt: new Date()
              }
            }
          },
          { upsert: true, returnDocument: 'after' }
        );

      return result;
    },

    createOrder: async (parent, { shippingAddress, paymentMethod }, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      // Récupérer panier
      const cart = await context.db.collection('carts')
        .findOne({ userId: context.userId });

      if (!cart || cart.items.length === 0) {
        throw new Error('Cart is empty');
      }

      // Calculer total
      const total = cart.items.reduce((sum, item) => sum + item.subtotal, 0);

      // Créer commande
      const order = {
        orderNumber: `ORD-${Date.now()}`,
        userId: context.userId,
        items: cart.items,
        status: 'PENDING',
        total,
        shippingAddress,
        paymentMethod,
        createdAt: new Date(),
        updatedAt: new Date()
      };

      const result = await context.db.collection('orders')
        .insertOne(order);

      // Vider panier
      await context.db.collection('carts')
        .updateOne(
          { userId: context.userId },
          { $set: { items: [] } }
        );

      // Publier événement (subscription)
      context.pubsub.publish('ORDER_CREATED', {
        orderUpdated: { ...order, _id: result.insertedId }
      });

      return { ...order, _id: result.insertedId };
    },

    createReview: async (parent, args, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      const { productId, rating, title, content } = args;

      // Vérifier que l'utilisateur a acheté le produit
      const hasPurchased = await context.db.collection('orders')
        .findOne({
          userId: context.userId,
          'items.productId': productId,
          status: 'DELIVERED'
        });

      if (!hasPurchased) {
        throw new Error('You can only review purchased products');
      }

      const review = {
        productId,
        authorId: context.userId,
        rating,
        title,
        content,
        helpful: 0,
        createdAt: new Date()
      };

      const result = await context.db.collection('reviews')
        .insertOne(review);

      // Mettre à jour rating moyen du produit
      await this.updateProductRating(context.db, productId);

      return { ...review, _id: result.insertedId };
    }
  }
};

// Fonction helper
async function updateProductRating(db, productId) {
  const stats = await db.collection('reviews')
    .aggregate([
      { $match: { productId } },
      {
        $group: {
          _id: null,
          avgRating: { $avg: '$rating' },
          count: { $sum: 1 }
        }
      }
    ])
    .toArray();

  if (stats.length > 0) {
    await db.collection('products')
      .updateOne(
        { _id: productId },
        {
          $set: {
            averageRating: stats[0].avgRating,
            reviewCount: stats[0].count
          }
        }
      );
  }
}
```

### Cas 2 : Subscriptions temps réel

```javascript
const { createServer } = require('http');
const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');
const { ApolloServerPluginDrainHttpServer } = require('@apollo/server/plugin/drainHttpServer');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const { WebSocketServer } = require('ws');
const { useServer } = require('graphql-ws/lib/use/ws');
const { PubSub } = require('graphql-subscriptions');
const express = require('express');
const bodyParser = require('body-parser');

const pubsub = new PubSub();

const typeDefs = `#graphql
  type Message {
    id: ObjectID!
    content: String!
    author: User!
    room: ChatRoom!
    createdAt: DateTime!
  }

  type ChatRoom {
    id: ObjectID!
    name: String!
    members: [User!]!
    messages: [Message!]!
    lastMessage: Message
    unreadCount(userId: ObjectID!): Int!
  }

  type User {
    id: ObjectID!
    name: String!
    status: UserStatus!
    lastSeen: DateTime
  }

  enum UserStatus {
    ONLINE
    AWAY
    OFFLINE
  }

  type Query {
    chatRoom(id: ObjectID!): ChatRoom
    chatRooms: [ChatRoom!]!
    messages(roomId: ObjectID!, limit: Int = 50): [Message!]!
  }

  type Mutation {
    sendMessage(roomId: ObjectID!, content: String!): Message!
    createChatRoom(name: String!, memberIds: [ObjectID!]!): ChatRoom!
    joinChatRoom(roomId: ObjectID!): ChatRoom!
    leaveChatRoom(roomId: ObjectID!): Boolean!
    updateUserStatus(status: UserStatus!): User!
    markAsRead(roomId: ObjectID!): Boolean!
  }

  type Subscription {
    messageAdded(roomId: ObjectID!): Message!
    userStatusChanged(userId: ObjectID!): User!
    typingIndicator(roomId: ObjectID!): TypingIndicator!
  }

  type TypingIndicator {
    userId: ObjectID!
    userName: String!
    isTyping: Boolean!
  }
`;

const resolvers = {
  Query: {
    chatRoom: async (parent, { id }, context) => {
      return await context.db.collection('chat_rooms')
        .findOne({ _id: id });
    },

    chatRooms: async (parent, args, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      return await context.db.collection('chat_rooms')
        .find({ memberIds: context.userId })
        .toArray();
    },

    messages: async (parent, { roomId, limit }, context) => {
      return await context.db.collection('messages')
        .find({ roomId })
        .sort({ createdAt: -1 })
        .limit(limit)
        .toArray();
    }
  },

  ChatRoom: {
    id: (parent) => parent._id,

    members: async (parent, args, context) => {
      return await context.db.collection('users')
        .find({ _id: { $in: parent.memberIds } })
        .toArray();
    },

    messages: async (parent, args, context) => {
      return await context.db.collection('messages')
        .find({ roomId: parent._id })
        .sort({ createdAt: -1 })
        .limit(50)
        .toArray();
    },

    lastMessage: async (parent, args, context) => {
      return await context.db.collection('messages')
        .findOne(
          { roomId: parent._id },
          { sort: { createdAt: -1 } }
        );
    },

    unreadCount: async (parent, { userId }, context) => {
      const user = await context.db.collection('users')
        .findOne({ _id: userId });

      if (!user) return 0;

      const lastRead = user.lastReadByRoom?.[parent._id.toString()];

      if (!lastRead) {
        return await context.db.collection('messages')
          .countDocuments({ roomId: parent._id });
      }

      return await context.db.collection('messages')
        .countDocuments({
          roomId: parent._id,
          createdAt: { $gt: lastRead }
        });
    }
  },

  Message: {
    id: (parent) => parent._id,

    author: async (parent, args, context) => {
      return await context.loaders.user.load(parent.authorId);
    },

    room: async (parent, args, context) => {
      return await context.db.collection('chat_rooms')
        .findOne({ _id: parent.roomId });
    }
  },

  Mutation: {
    sendMessage: async (parent, { roomId, content }, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      // Vérifier que l'utilisateur est membre
      const room = await context.db.collection('chat_rooms')
        .findOne({
          _id: roomId,
          memberIds: context.userId
        });

      if (!room) {
        throw new Error('Not a member of this room');
      }

      // Créer message
      const message = {
        content,
        authorId: context.userId,
        roomId,
        createdAt: new Date()
      };

      const result = await context.db.collection('messages')
        .insertOne(message);

      const fullMessage = {
        ...message,
        _id: result.insertedId
      };

      // Publier via subscription
      pubsub.publish(`MESSAGE_ADDED_${roomId}`, {
        messageAdded: fullMessage
      });

      return fullMessage;
    },

    createChatRoom: async (parent, { name, memberIds }, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      // Ajouter le créateur aux membres
      if (!memberIds.includes(context.userId)) {
        memberIds.push(context.userId);
      }

      const room = {
        name,
        memberIds,
        createdBy: context.userId,
        createdAt: new Date()
      };

      const result = await context.db.collection('chat_rooms')
        .insertOne(room);

      return { ...room, _id: result.insertedId };
    },

    updateUserStatus: async (parent, { status }, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      const user = await context.db.collection('users')
        .findOneAndUpdate(
          { _id: context.userId },
          {
            $set: {
              status,
              lastSeen: new Date()
            }
          },
          { returnDocument: 'after' }
        );

      // Publier changement de statut
      pubsub.publish(`USER_STATUS_${context.userId}`, {
        userStatusChanged: user
      });

      return user;
    },

    markAsRead: async (parent, { roomId }, context) => {
      if (!context.userId) {
        throw new Error('Not authenticated');
      }

      await context.db.collection('users')
        .updateOne(
          { _id: context.userId },
          {
            $set: {
              [`lastReadByRoom.${roomId}`]: new Date()
            }
          }
        );

      return true;
    }
  },

  Subscription: {
    messageAdded: {
      subscribe: (parent, { roomId }, context) => {
        // S'abonner aux nouveaux messages dans cette room
        return pubsub.asyncIterator(`MESSAGE_ADDED_${roomId}`);
      }
    },

    userStatusChanged: {
      subscribe: (parent, { userId }, context) => {
        return pubsub.asyncIterator(`USER_STATUS_${userId}`);
      }
    },

    typingIndicator: {
      subscribe: (parent, { roomId }, context) => {
        return pubsub.asyncIterator(`TYPING_${roomId}`);
      }
    }
  }
};

// Configuration serveur avec WebSocket
async function startServerWithSubscriptions() {
  const app = express();
  const httpServer = createServer(app);

  const schema = makeExecutableSchema({ typeDefs, resolvers });

  // WebSocket server pour subscriptions
  const wsServer = new WebSocketServer({
    server: httpServer,
    path: '/graphql'
  });

  const serverCleanup = useServer(
    {
      schema,
      context: async (ctx) => {
        // Extraire token depuis connexion params
        const token = ctx.connectionParams?.authentication;
        const userId = await authenticateToken(token);

        return {
          db,
          userId,
          pubsub
        };
      }
    },
    wsServer
  );

  // Apollo Server
  const server = new ApolloServer({
    schema,
    plugins: [
      ApolloServerPluginDrainHttpServer({ httpServer }),
      {
        async serverWillStart() {
          return {
            async drainServer() {
              await serverCleanup.dispose();
            }
          };
        }
      }
    ]
  });

  await server.start();

  app.use(
    '/graphql',
    bodyParser.json(),
    expressMiddleware(server, {
      context: async ({ req }) => {
        const token = req.headers.authorization?.split(' ')[1];
        const userId = await authenticateToken(token);

        return {
          db,
          userId,
          pubsub,
          loaders: createLoaders(db)
        };
      }
    })
  );

  httpServer.listen(4000, () => {
    console.log('🚀 Server ready at http://localhost:4000/graphql');
    console.log('🔌 Subscriptions ready at ws://localhost:4000/graphql');
  });
}

startServerWithSubscriptions();
```

### Client-side subscription (exemple React)

```javascript
import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';
import { useSubscription, gql } from '@apollo/client';

// Configuration Apollo Client
const httpLink = new HttpLink({
  uri: 'http://localhost:4000/graphql'
});

const wsLink = new GraphQLWsLink(
  createClient({
    url: 'ws://localhost:4000/graphql',
    connectionParams: {
      authentication: localStorage.getItem('token')
    }
  })
);

// Split selon type d'opération
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  httpLink
);

const client = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache()
});

// Composant React avec subscription
const MESSAGE_ADDED = gql`
  subscription MessageAdded($roomId: ObjectID!) {
    messageAdded(roomId: $roomId) {
      id
      content
      author {
        id
        name
      }
      createdAt
    }
  }
`;

function ChatRoom({ roomId }) {
  const { data, loading } = useSubscription(MESSAGE_ADDED, {
    variables: { roomId }
  });

  if (loading) return <div>Loading...</div>;

  if (data) {
    console.log('New message:', data.messageAdded);
    // Mettre à jour UI avec nouveau message
  }

  return <div>Chat messages...</div>;
}
```

### Cas 3 : GraphQL Federation (Microservices)

```javascript
// Service 1: Users Service
const { ApolloServer } = require('@apollo/server');
const { buildSubgraphSchema } = require('@apollo/subgraph');
const { gql } = require('graphql-tag');

const typeDefs = gql`
  type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
    createdAt: DateTime!
  }

  extend type Post @key(fields: "id") {
    id: ID! @external
    author: User!
  }

  extend type Comment @key(fields: "id") {
    id: ID! @external
    author: User!
  }

  type Query {
    user(id: ID!): User
    users: [User!]!
  }
`;

const resolvers = {
  User: {
    __resolveReference: async (reference, context) => {
      return await context.db.collection('users')
        .findOne({ _id: new ObjectId(reference.id) });
    }
  },

  Post: {
    author: async (parent, args, context) => {
      return await context.db.collection('users')
        .findOne({ _id: parent.authorId });
    }
  },

  Comment: {
    author: async (parent, args, context) => {
      return await context.db.collection('users')
        .findOne({ _id: parent.authorId });
    }
  },

  Query: {
    user: async (parent, { id }, context) => {
      return await context.db.collection('users')
        .findOne({ _id: new ObjectId(id) });
    },

    users: async (parent, args, context) => {
      return await context.db.collection('users')
        .find()
        .toArray();
    }
  }
};

// Service 2: Posts Service
const postsTypeDefs = gql`
  type Post @key(fields: "id") {
    id: ID!
    title: String!
    content: String!
    authorId: ID!
    published: Boolean!
    createdAt: DateTime!
  }

  extend type User @key(fields: "id") {
    id: ID! @external
    posts: [Post!]!
  }

  type Query {
    post(id: ID!): Post
    posts(published: Boolean): [Post!]!
  }

  type Mutation {
    createPost(
      title: String!
      content: String!
      authorId: ID!
    ): Post!
  }
`;

const postsResolvers = {
  Post: {
    __resolveReference: async (reference, context) => {
      return await context.db.collection('posts')
        .findOne({ _id: new ObjectId(reference.id) });
    }
  },

  User: {
    posts: async (parent, args, context) => {
      return await context.db.collection('posts')
        .find({ authorId: new ObjectId(parent.id) })
        .toArray();
    }
  },

  Query: {
    post: async (parent, { id }, context) => {
      return await context.db.collection('posts')
        .findOne({ _id: new ObjectId(id) });
    },

    posts: async (parent, { published }, context) => {
      const filter = published !== undefined ? { published } : {};
      return await context.db.collection('posts')
        .find(filter)
        .toArray();
    }
  },

  Mutation: {
    createPost: async (parent, args, context) => {
      const { title, content, authorId } = args;

      const post = {
        title,
        content,
        authorId: new ObjectId(authorId),
        published: false,
        createdAt: new Date()
      };

      const result = await context.db.collection('posts')
        .insertOne(post);

      return { ...post, _id: result.insertedId };
    }
  }
};

// Gateway (Apollo Router)
// Configuration YAML
/*
supergraph:
  listen: 0.0.0.0:4000

subgraphs:
  users:
    routing_url: http://localhost:4001/graphql
    schema:
      file: ./users-schema.graphql

  posts:
    routing_url: http://localhost:4002/graphql
    schema:
      file: ./posts-schema.graphql
*/

// Le gateway compose automatiquement le schéma complet
// et route les requêtes aux bons services
```

---

## Authentification et autorisation

### JWT Authentication

```javascript
const jwt = require('jsonwebtoken');

// Middleware d'authentification
async function authenticateToken(token) {
  if (!token) return null;

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return decoded.userId;
  } catch (error) {
    return null;
  }
}

// Context avec authentification
const server = new ApolloServer({
  typeDefs,
  resolvers
});

await startStandaloneServer(server, {
  context: async ({ req }) => {
    const token = req.headers.authorization?.split(' ')[1];
    const userId = await authenticateToken(token);

    return {
      db,
      userId,
      loaders: createLoaders(db)
    };
  }
});
```

### Directives d'autorisation

```javascript
const { mapSchema, getDirective, MapperKind } = require('@graphql-tools/utils');
const { defaultFieldResolver } = require('graphql');

// Définir directive
const typeDefs = `
  directive @auth(requires: Role = USER) on FIELD_DEFINITION | OBJECT

  enum Role {
    ADMIN
    USER
    GUEST
  }

  type Query {
    users: [User!]! @auth(requires: ADMIN)
    me: User @auth
    posts: [Post!]!
  }

  type Mutation {
    deleteUser(id: ID!): Boolean! @auth(requires: ADMIN)
    updateProfile(name: String!): User! @auth
  }
`;

// Implémentation directive
function authDirective(directiveName = 'auth') {
  return {
    authDirectiveTypeDefs: `directive @${directiveName}(
      requires: Role = USER
    ) on FIELD_DEFINITION | OBJECT`,

    authDirectiveTransformer: (schema) => {
      return mapSchema(schema, {
        [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
          const authDirective = getDirective(
            schema,
            fieldConfig,
            directiveName
          )?.[0];

          if (authDirective) {
            const { requires } = authDirective;
            const { resolve = defaultFieldResolver } = fieldConfig;

            fieldConfig.resolve = async function (source, args, context, info) {
              // Vérifier authentification
              if (!context.userId) {
                throw new Error('Not authenticated');
              }

              // Vérifier rôle si nécessaire
              if (requires) {
                const user = await context.db.collection('users')
                  .findOne({ _id: context.userId });

                if (!user || user.role !== requires) {
                  throw new Error(
                    `Requires ${requires} role`
                  );
                }
              }

              return resolve(source, args, context, info);
            };
          }

          return fieldConfig;
        }
      });
    }
  };
}
```

---

## Performance et bonnes pratiques

### Pagination cursor-based

```javascript
// ✅ Utiliser cursor-based pagination pour grandes collections
const typeDefs = `
  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type ProductEdge {
    node: Product!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  type Query {
    products(first: Int, after: String): ProductConnection!
  }
`;
```

### Query Complexity Analysis

```javascript
const { createComplexityPlugin } = require('@escape.tech/graphql-armor-max-depth');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    createComplexityPlugin({
      maxComplexity: 1000,
      objectCost: 1,
      scalarCost: 0,
      depthCostFactor: 1.5,
      onComplete: (complexity) => {
        console.log('Query complexity:', complexity);
      }
    })
  ]
});
```

### Caching

```javascript
// Utiliser Redis pour cache
const Redis = require('ioredis');
const redis = new Redis();

const resolvers = {
  Query: {
    user: async (parent, { id }, context) => {
      // Chercher dans cache
      const cacheKey = `user:${id}`;
      const cached = await redis.get(cacheKey);

      if (cached) {
        return JSON.parse(cached);
      }

      // Requête DB
      const user = await context.db.collection('users')
        .findOne({ _id: new ObjectId(id) });

      // Mettre en cache (5 minutes)
      if (user) {
        await redis.setex(cacheKey, 300, JSON.stringify(user));
      }

      return user;
    }
  }
};
```

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Utiliser DataLoader pour éviter N+1
const loaders = {
  user: new DataLoader(batchLoadUsers),
  posts: new DataLoader(batchLoadPosts)
};

// 2. Pagination pour grandes listes
products(first: 20, after: "cursor") {
  edges {
    node { ... }
  }
}

// 3. Limiter profondeur de requêtes
plugins: [
  createMaxDepthPlugin({ maxDepth: 10 })
]

// 4. Authentification dans context
context: async ({ req }) => ({
  userId: await authenticateToken(req.headers.authorization)
})

// 5. Validation des inputs
input CreateUserInput {
  name: String! @constraint(minLength: 2, maxLength: 100)
  email: String! @constraint(format: "email")
}
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas faire de requêtes dans boucle
// ❌ N+1 problem
User: {
  posts: async (parent) => {
    return await db.collection('posts')
      .find({ authorId: parent._id })
      .toArray();
  }
}

// 2. Ne pas exposer données sensibles
// ❌ Expose password hash
type User {
  id: ID!
  email: String!
  passwordHash: String!  // DANGER!
}

// 3. Ne pas permettre requêtes illimitées
// ❌ Peut charger 1 million de produits
type Query {
  products: [Product!]!  // Pas de limit!
}

// 4. Ne pas ignorer validation
// ❌ Injection possible
mutation {
  updateUser(id: "'; DROP TABLE users;--")
}

// 5. Ne pas logger requêtes complètes
// ❌ Peut contenir mots de passe
console.log('Query:', req.body.query);
```

---

## Monitoring et debugging

### Apollo Studio Integration

```javascript
const { ApolloServer } = require('@apollo/server');
const { ApolloServerPluginUsageReporting } = require('@apollo/server/plugin/usageReporting');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    ApolloServerPluginUsageReporting({
      sendVariableValues: { all: true },
      sendHeaders: { all: true }
    })
  ]
});
```

### Custom logging

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      async requestDidStart() {
        const start = Date.now();

        return {
          async willSendResponse({ operation }) {
            const duration = Date.now() - start;

            console.log({
              operation: operation?.operation,
              name: operation?.name?.value,
              duration: `${duration}ms`
            });
          },

          async didEncounterErrors({ errors }) {
            errors.forEach(error => {
              console.error('GraphQL Error:', {
                message: error.message,
                path: error.path,
                extensions: error.extensions
              });
            });
          }
        };
      }
    }
  ]
});
```

---

## Conclusion

GraphQL avec MongoDB offre :
- ✅ **Flexibilité client-side** (requêtes personnalisées)
- ✅ **Typage fort** (validation automatique)
- ✅ **Performance** (DataLoader, batching)
- ✅ **Temps réel** (subscriptions WebSocket)
- ✅ **Microservices** (Federation)
- ✅ **DX exceptionnelle** (introspection, playground)

**Points clés à retenir :**
1. DataLoader obligatoire pour éviter N+1
2. Pagination cursor-based pour scalabilité
3. Authentification/autorisation via context
4. Monitoring avec Apollo Studio
5. Subscriptions pour temps réel

**Cas d'usage idéaux :**
- Applications mobiles/web modernes
- Plateformes avec multiples clients
- Microservices architecture
- Applications temps réel (chat, notifications)
- Dashboards avec données complexes

**Technologies complémentaires :**
- Apollo Server/Client
- DataLoader pour batching
- Redis pour caching
- Apollo Studio pour monitoring

---


⏭️ [Performance et Tuning](/17-performance-tuning/README.md)
