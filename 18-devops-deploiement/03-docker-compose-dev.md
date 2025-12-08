🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.3 Docker Compose pour Environnements de Développement

## Introduction

Docker Compose est l'outil de prédilection pour créer des environnements de développement MongoDB reproductibles, isolés et rapidement déployables. Cette section se concentre sur les patterns et configurations optimisés pour maximiser la productivité des développeurs tout en maintenant la parité avec les environnements de production.

Un bon environnement de développement Docker doit offrir :
- **Rapidité** : Démarrage instantané et hot-reload
- **Isolation** : Pas de conflits entre projets
- **Reproductibilité** : Même environnement pour toute l'équipe
- **Debugging** : Outils de débogage intégrés
- **Performance** : Pas de ralentissements perceptibles

---

## Philosophie des Environnements de Développement

### Dev/Prod Parity

```
┌─────────────────────────────────────────────────────────────────┐
│                    Dev/Prod Parity Spectrum                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Development          Staging            Production             │
│  ┌──────────────┐    ┌──────────────┐   ┌──────────────┐        │
│  │ Optimized    │    │ Production   │   │ Production   │        │
│  │ for Speed    │    │ Configuration│   │ Configuration│        │
│  │              │    │              │   │              │        │
│  │ • Fast start │    │ • Replica Set│   │ • Replica Set│        │
│  │ • Hot reload │    │ • Auth       │   │ • Auth       │        │
│  │ • Debug logs │    │ • Monitoring │   │ • Monitoring │        │
│  │ • No auth    │◀───│ • TLS        │◀──│ • TLS        │        │
│  │ • Seed data  │    │ • Backups    │   │ • Backups    │        │
│  └──────────────┘    └──────────────┘   └──────────────┘        │
│       ▲                     ▲                   ▲               │
│       │                     │                   │               │
│       └─────────────────────┴───────────────────┘               │
│              Progressive Configuration                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Principes :**
1. **Développement** : Optimisé pour la vitesse et le feedback rapide
2. **Staging** : Configuration identique à la production
3. **Production** : Sécurisé, monitoré, hautement disponible

### Architecture Type d'un Environnement de Dev

```
┌────────────────────────────────────────────────────────────────┐
│              Development Stack Architecture                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Developer Machine                     │  │
│  │                                                          │  │
│  │  IDE/Editor          Browser              Terminal       │  │
│  │  ┌────────┐         ┌────────┐           ┌────────┐      │  │
│  │  │ VSCode │◀────────│ Chrome │◀──────────│  CLI   │      │  │
│  │  └───┬────┘         └───┬────┘           └───┬────┘      │  │
│  └──────┼──────────────────┼────────────────────┼───────────┘  │
│         │                  │                    │              │
│         ▼                  ▼                    ▼              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Docker Compose Network                      │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │   MongoDB   │  │     API     │  │   Frontend  │       │  │
│  │  │             │  │   Backend   │  │     App     │       │  │
│  │  │ • Port 27017│◀─│ • Hot Reload│◀─│ • Hot Reload│       │  │
│  │  │ • Volume    │  │ • Debug Port│  │ • Dev Server│       │  │
│  │  │ • Seed Data │  │ • Env Vars  │  │ • Proxy     │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  │         │                 │                 │            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │  Mongo      │  │   Redis     │  │  MailHog    │       │  │
│  │  │  Express    │  │   Cache     │  │  SMTP       │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  │                                                          │  │
│  │  Volumes:                                                │  │
│  │  • ./src:/app/src         (Code source)                  │  │
│  │  • ./data/mongo:/data/db  (Persistance)                  │  │
│  │  • mongo_data:/data/db    (Named volume)                 │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Configuration de Base pour Développement

### Stack Minimal : MongoDB + Application

```yaml
# docker-compose.yml
version: '3.9'

services:
  # MongoDB pour développement
  mongodb:
    image: mongo:7.0.5
    container_name: dev-mongodb

    # Port exposé pour connexion locale
    ports:
      - "27017:27017"

    # Configuration minimale sans authentification
    environment:
      # Pas d'auth en dev pour faciliter les tests
      MONGO_INITDB_DATABASE: dev_db

    # Volume pour persistance des données entre redémarrages
    volumes:
      # Données persistantes
      - mongodb_data:/data/db

      # Scripts d'initialisation (seed data)
      - ./mongo-init:/docker-entrypoint-initdb.d:ro

      # Configuration custom si nécessaire
      - ./config/mongod-dev.conf:/etc/mongod.conf:ro

    # Configuration optimisée pour le développement
    command:
      - --storageEngine=wiredTiger
      - --wiredTigerCacheSizeGB=0.5  # Limité pour ne pas saturer la RAM
      - --noauth                      # Pas d'auth en dev
      - --quiet                       # Moins de logs verbeux

    # Health check simple
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s

    # Restart automatique
    restart: unless-stopped

    networks:
      - dev_network

  # Application backend (exemple Node.js)
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev

    container_name: dev-api

    ports:
      - "3000:3000"
      - "9229:9229"  # Port debug Node.js

    environment:
      NODE_ENV: development
      MONGODB_URI: mongodb://mongodb:27017/dev_db
      DEBUG: "app:*"
      LOG_LEVEL: debug

    # Hot reload avec volumes
    volumes:
      - ./backend/src:/app/src:delegated
      - ./backend/package.json:/app/package.json:ro
      - /app/node_modules  # Anonymous volume pour node_modules

    # Commande de développement avec nodemon
    command: npm run dev

    depends_on:
      mongodb:
        condition: service_healthy

    networks:
      - dev_network

volumes:
  mongodb_data:
    driver: local

networks:
  dev_network:
    driver: bridge
```

```yaml
# config/mongod-dev.conf
storage:
  dbPath: /data/db
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.5
    collectionConfig:
      blockCompressor: snappy

systemLog:
  quiet: true
  destination: file
  path: /data/db/mongod.log
  logAppend: true

net:
  port: 27017
  bindIp: 0.0.0.0

operationProfiling:
  mode: all  # Profile toutes les opérations en dev
  slowOpThresholdMs: 50
```

### Makefile pour Simplifier les Commandes

```makefile
# Makefile
.PHONY: help start stop restart logs shell seed clean test

# Variables
COMPOSE := docker-compose
COMPOSE_DEV := $(COMPOSE) -f docker-compose.yml
COMPOSE_TEST := $(COMPOSE) -f docker-compose.test.yml

help: ## Affiche l'aide
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

start: ## Démarre l'environnement de développement
	$(COMPOSE_DEV) up -d
	@echo "✅ Environnement démarré"
	@echo "📊 MongoDB: mongodb://localhost:27017"
	@echo "🚀 API: http://localhost:3000"
	@$(MAKE) logs

stop: ## Arrête l'environnement
	$(COMPOSE_DEV) down
	@echo "✅ Environnement arrêté"

restart: stop start ## Redémarre l'environnement

logs: ## Affiche les logs
	$(COMPOSE_DEV) logs -f

logs-mongo: ## Logs MongoDB uniquement
	$(COMPOSE_DEV) logs -f mongodb

shell: ## Ouvre un shell dans le conteneur MongoDB
	$(COMPOSE_DEV) exec mongodb mongosh

shell-api: ## Ouvre un shell dans le conteneur API
	$(COMPOSE_DEV) exec api /bin/sh

seed: ## Charge les données de test
	@echo "🌱 Chargement des données de seed..."
	$(COMPOSE_DEV) exec -T mongodb mongosh dev_db < ./mongo-init/seed.js
	@echo "✅ Données chargées"

clean: ## Nettoie l'environnement (ATTENTION: supprime les données)
	$(COMPOSE_DEV) down -v
	@echo "⚠️  Données supprimées"

rebuild: clean ## Rebuild les images et redémarre
	$(COMPOSE_DEV) build --no-cache
	$(MAKE) start

test: ## Lance les tests
	$(COMPOSE_TEST) up --abort-on-container-exit --exit-code-from tests
	$(COMPOSE_TEST) down -v

ps: ## Affiche le statut des conteneurs
	$(COMPOSE_DEV) ps

stats: ## Affiche les statistiques des conteneurs
	docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

backup: ## Backup de la base de données
	@mkdir -p ./backups
	@echo "💾 Backup en cours..."
	$(COMPOSE_DEV) exec -T mongodb mongodump --out=/tmp/backup --gzip
	docker cp dev-mongodb:/tmp/backup ./backups/backup-$$(date +%Y%m%d-%H%M%S)
	@echo "✅ Backup terminé"

restore: ## Restore le dernier backup
	@if [ -z "$$(ls -t ./backups | head -1)" ]; then \
		echo "❌ Aucun backup trouvé"; \
		exit 1; \
	fi
	@echo "📂 Restoration du backup: $$(ls -t ./backups | head -1)"
	docker cp ./backups/$$(ls -t ./backups | head -1) dev-mongodb:/tmp/restore
	$(COMPOSE_DEV) exec -T mongodb mongorestore /tmp/restore --gzip --drop
	@echo "✅ Restoration terminée"
```

### Scripts d'Initialisation et Seed Data

```javascript
// mongo-init/01-init.js
// Script exécuté au premier démarrage de MongoDB

print('🚀 Initialisation de la base de données de développement...');

// Créer la base de données
db = db.getSiblingDB('dev_db');

// Créer les collections
db.createCollection('users');
db.createCollection('products');
db.createCollection('orders');

// Créer les index
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ createdAt: -1 });
db.products.createIndex({ name: 'text', description: 'text' });
db.products.createIndex({ category: 1, price: 1 });
db.orders.createIndex({ userId: 1, createdAt: -1 });
db.orders.createIndex({ status: 1 });

print('✅ Base de données initialisée avec succès');
```

```javascript
// mongo-init/02-seed.js
// Données de test pour le développement

print('🌱 Chargement des données de seed...');

db = db.getSiblingDB('dev_db');

// Users
const users = [
  {
    _id: ObjectId('507f1f77bcf86cd799439011'),
    email: 'john.doe@example.com',
    name: 'John Doe',
    role: 'user',
    createdAt: new Date('2024-01-01T00:00:00Z'),
    updatedAt: new Date('2024-01-01T00:00:00Z')
  },
  {
    _id: ObjectId('507f1f77bcf86cd799439012'),
    email: 'jane.smith@example.com',
    name: 'Jane Smith',
    role: 'admin',
    createdAt: new Date('2024-01-02T00:00:00Z'),
    updatedAt: new Date('2024-01-02T00:00:00Z')
  },
  {
    _id: ObjectId('507f1f77bcf86cd799439013'),
    email: 'bob.wilson@example.com',
    name: 'Bob Wilson',
    role: 'user',
    createdAt: new Date('2024-01-03T00:00:00Z'),
    updatedAt: new Date('2024-01-03T00:00:00Z')
  }
];

db.users.insertMany(users);
print(`✅ ${users.length} utilisateurs créés`);

// Products
const products = [
  {
    _id: ObjectId('607f1f77bcf86cd799439021'),
    name: 'Laptop Pro 15"',
    description: 'High-performance laptop for developers',
    category: 'Electronics',
    price: 1299.99,
    stock: 50,
    tags: ['laptop', 'computer', 'development'],
    createdAt: new Date('2024-01-01T00:00:00Z')
  },
  {
    _id: ObjectId('607f1f77bcf86cd799439022'),
    name: 'Mechanical Keyboard',
    description: 'RGB mechanical keyboard with Cherry MX switches',
    category: 'Electronics',
    price: 149.99,
    stock: 120,
    tags: ['keyboard', 'peripherals'],
    createdAt: new Date('2024-01-02T00:00:00Z')
  },
  {
    _id: ObjectId('607f1f77bcf86cd799439023'),
    name: 'Ergonomic Mouse',
    description: 'Wireless ergonomic mouse',
    category: 'Electronics',
    price: 79.99,
    stock: 200,
    tags: ['mouse', 'peripherals', 'wireless'],
    createdAt: new Date('2024-01-03T00:00:00Z')
  }
];

db.products.insertMany(products);
print(`✅ ${products.length} produits créés`);

// Orders
const orders = [
  {
    _id: ObjectId('707f1f77bcf86cd799439031'),
    userId: ObjectId('507f1f77bcf86cd799439011'),
    items: [
      {
        productId: ObjectId('607f1f77bcf86cd799439021'),
        quantity: 1,
        price: 1299.99
      },
      {
        productId: ObjectId('607f1f77bcf86cd799439022'),
        quantity: 1,
        price: 149.99
      }
    ],
    totalAmount: 1449.98,
    status: 'completed',
    createdAt: new Date('2024-01-10T10:00:00Z'),
    completedAt: new Date('2024-01-15T14:30:00Z')
  },
  {
    _id: ObjectId('707f1f77bcf86cd799439032'),
    userId: ObjectId('507f1f77bcf86cd799439013'),
    items: [
      {
        productId: ObjectId('607f1f77bcf86cd799439023'),
        quantity: 2,
        price: 79.99
      }
    ],
    totalAmount: 159.98,
    status: 'pending',
    createdAt: new Date('2024-01-20T15:00:00Z')
  }
];

db.orders.insertMany(orders);
print(`✅ ${orders.length} commandes créées`);

// Afficher les statistiques
print('\n📊 Statistiques:');
print(`- Utilisateurs: ${db.users.countDocuments()}`);
print(`- Produits: ${db.products.countDocuments()}`);
print(`- Commandes: ${db.orders.countDocuments()}`);

print('\n✨ Seed data chargé avec succès!');
```

---

## Stack Complète de Développement

### Docker Compose Multi-Service

```yaml
# docker-compose.full.yml
version: '3.9'

# Configuration partagée entre services
x-node-common: &node-common
  image: node:20-alpine
  working_dir: /app
  volumes:
    - /app/node_modules
  environment: &node-env
    NODE_ENV: development
    LOG_LEVEL: debug
  networks:
    - dev_network

services:
  #############################
  # MongoDB                   #
  #############################

  mongodb:
    image: mongo:7.0.5
    container_name: dev-mongodb
    hostname: mongodb

    ports:
      - "27017:27017"

    environment:
      MONGO_INITDB_DATABASE: dev_db

    volumes:
      - mongodb_data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
      - ./config/mongod-dev.conf:/etc/mongod.conf:ro

    command:
      - --config=/etc/mongod.conf
      - --noauth

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 3

    networks:
      - dev_network

  #############################
  # Mongo Express (UI)        #
  #############################

  mongo-express:
    image: mongo-express:1.0.0
    container_name: dev-mongo-express

    ports:
      - "8081:8081"

    environment:
      ME_CONFIG_MONGODB_URL: mongodb://mongodb:27017/
      ME_CONFIG_BASICAUTH: false
      ME_CONFIG_MONGODB_ENABLE_ADMIN: true

    depends_on:
      mongodb:
        condition: service_healthy

    networks:
      - dev_network

  #############################
  # Redis (Cache)             #
  #############################

  redis:
    image: redis:7.2-alpine
    container_name: dev-redis

    ports:
      - "6379:6379"

    command: redis-server --appendonly yes --requirepass devpassword

    volumes:
      - redis_data:/data

    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

    networks:
      - dev_network

  #############################
  # Backend API               #
  #############################

  api:
    <<: *node-common
    container_name: dev-api

    build:
      context: ./backend
      dockerfile: Dockerfile.dev
      args:
        NODE_VERSION: 20

    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port

    environment:
      <<: *node-env
      PORT: 3000
      MONGODB_URI: mongodb://mongodb:27017/dev_db
      REDIS_URL: redis://:devpassword@redis:6379
      JWT_SECRET: dev-secret-key
      CORS_ORIGIN: http://localhost:3001

    volumes:
      - ./backend/src:/app/src:delegated
      - ./backend/package.json:/app/package.json:ro
      - ./backend/package-lock.json:/app/package-lock.json:ro
      - api_node_modules:/app/node_modules

    command: npm run dev

    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy

    networks:
      - dev_network

  #############################
  # Frontend                  #
  #############################

  frontend:
    <<: *node-common
    container_name: dev-frontend

    build:
      context: ./frontend
      dockerfile: Dockerfile.dev

    ports:
      - "3001:3001"

    environment:
      <<: *node-env
      PORT: 3001
      REACT_APP_API_URL: http://localhost:3000
      CHOKIDAR_USEPOLLING: true  # Pour le hot reload sur certains OS
      WATCHPACK_POLLING: true

    volumes:
      - ./frontend/src:/app/src:delegated
      - ./frontend/public:/app/public:delegated
      - ./frontend/package.json:/app/package.json:ro
      - frontend_node_modules:/app/node_modules

    command: npm start

    stdin_open: true
    tty: true

    depends_on:
      - api

    networks:
      - dev_network

  #############################
  # MailHog (SMTP Testing)    #
  #############################

  mailhog:
    image: mailhog/mailhog:v1.0.1
    container_name: dev-mailhog

    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI

    networks:
      - dev_network

  #############################
  # Nginx (Reverse Proxy)     #
  #############################

  nginx:
    image: nginx:1.25-alpine
    container_name: dev-nginx

    ports:
      - "80:80"

    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro

    depends_on:
      - api
      - frontend

    networks:
      - dev_network

  #############################
  # Adminer (DB Management)   #
  #############################

  adminer:
    image: adminer:4.8.1
    container_name: dev-adminer

    ports:
      - "8080:8080"

    environment:
      ADMINER_DEFAULT_SERVER: mongodb

    networks:
      - dev_network

volumes:
  mongodb_data:
    driver: local
  redis_data:
    driver: local
  api_node_modules:
  frontend_node_modules:

networks:
  dev_network:
    driver: bridge
    name: dev_network
```

```nginx
# nginx/conf.d/default.conf
upstream api_backend {
    server api:3000;
}

upstream frontend_server {
    server frontend:3001;
}

# Redirect HTTP to frontend
server {
    listen 80 default_server;
    server_name localhost;

    # Frontend
    location / {
        proxy_pass http://frontend_server;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        # WebSocket support for hot reload
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API
    location /api {
        rewrite ^/api/(.*)$ /$1 break;
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
```

---

## Dockerfiles Optimisés pour le Développement

### Backend Dockerfile.dev

```dockerfile
# backend/Dockerfile.dev
FROM node:20-alpine

# Install dependencies for native modules
RUN apk add --no-cache \
    python3 \
    make \
    g++ \
    bash \
    curl

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies with dev dependencies
RUN npm ci --include=dev

# Install nodemon globally for hot reload
RUN npm install -g nodemon

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy application code (will be overridden by volume in compose)
COPY . .

# Change ownership
RUN chown -R nodejs:nodejs /app

USER nodejs

# Expose ports
EXPOSE 3000 9229

# Start with nodemon for hot reload
CMD ["npm", "run", "dev"]
```

```json
// backend/package.json (extrait)
{
  "scripts": {
    "dev": "nodemon --inspect=0.0.0.0:9229 --watch src --exec node src/index.js",
    "test": "jest --watchAll",
    "lint": "eslint src/",
    "format": "prettier --write src/"
  },
  "nodemonConfig": {
    "watch": ["src"],
    "ext": "js,json",
    "ignore": ["src/**/*.test.js"],
    "delay": 500
  }
}
```

### Frontend Dockerfile.dev

```dockerfile
# frontend/Dockerfile.dev
FROM node:20-alpine

# Install dependencies
RUN apk add --no-cache bash

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Install development tools
RUN npm install -g serve

# Copy application (will be overridden by volume)
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

USER nodejs

EXPOSE 3001

# Start development server with hot reload
CMD ["npm", "start"]
```

---

## Environnements Multiples avec Profiles

### Docker Compose avec Profiles

```yaml
# docker-compose.profiles.yml
version: '3.9'

services:
  # Core services (toujours démarrés)
  mongodb:
    image: mongo:7.0.5
    # ... configuration de base

  api:
    # ... configuration de base
    profiles: ["backend", "full"]

  frontend:
    # ... configuration de base
    profiles: ["frontend", "full"]

  # Services optionnels
  mongo-express:
    image: mongo-express:1.0.0
    profiles: ["tools", "full"]
    # ... configuration

  redis:
    image: redis:7.2-alpine
    profiles: ["cache", "full"]
    # ... configuration

  mailhog:
    image: mailhog/mailhog
    profiles: ["mail", "full"]
    # ... configuration

  # Services de test
  test-runner:
    build:
      context: ./backend
      dockerfile: Dockerfile.test
    profiles: ["test"]
    command: npm test
    depends_on:
      - mongodb

  # Services de performance
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    profiles: ["monitoring"]
    # ... configuration

networks:
  dev_network:
    driver: bridge
```

```bash
#!/bin/bash
# scripts/dev.sh - Helper script pour démarrer différents profils

set -euo pipefail

PROFILE="${1:-full}"

case "$PROFILE" in
  minimal)
    echo "🚀 Démarrage profil minimal (MongoDB seulement)"
    docker-compose -f docker-compose.profiles.yml up -d mongodb
    ;;

  backend)
    echo "🚀 Démarrage profil backend (MongoDB + API)"
    docker-compose -f docker-compose.profiles.yml --profile backend up -d
    ;;

  frontend)
    echo "🚀 Démarrage profil frontend (MongoDB + API + Frontend)"
    docker-compose -f docker-compose.profiles.yml --profile frontend --profile backend up -d
    ;;

  full)
    echo "🚀 Démarrage profil complet"
    docker-compose -f docker-compose.profiles.yml --profile full up -d
    ;;

  test)
    echo "🧪 Exécution des tests"
    docker-compose -f docker-compose.profiles.yml --profile test up --abort-on-container-exit
    docker-compose -f docker-compose.profiles.yml --profile test down -v
    ;;

  *)
    echo "❌ Profil inconnu: $PROFILE"
    echo "Profils disponibles: minimal, backend, frontend, full, test"
    exit 1
    ;;
esac

echo "✅ Environnement démarré"
```

---

## Configuration par Environnement

### Variables d'Environnement avec .env

```bash
# .env.example
# Copier ce fichier vers .env et ajuster les valeurs

# Application
NODE_ENV=development
APP_NAME=MyApp
APP_VERSION=1.0.0
DEBUG=true

# MongoDB
MONGO_HOST=mongodb
MONGO_PORT=27017
MONGO_DATABASE=dev_db
MONGO_ROOT_USERNAME=
MONGO_ROOT_PASSWORD=

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=devpassword
REDIS_DB=0

# API
API_PORT=3000
API_DEBUG_PORT=9229
JWT_SECRET=dev-secret-key-change-in-prod
JWT_EXPIRES_IN=7d
CORS_ORIGIN=http://localhost:3001

# Frontend
FRONTEND_PORT=3001
REACT_APP_API_URL=http://localhost:3000

# Email (MailHog)
SMTP_HOST=mailhog
SMTP_PORT=1025
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=noreply@example.com

# Monitoring
PROMETHEUS_ENABLED=false
GRAFANA_ENABLED=false

# Logging
LOG_LEVEL=debug
LOG_FORMAT=pretty

# Feature Flags
FEATURE_NEW_UI=true
FEATURE_ANALYTICS=false
```

```yaml
# docker-compose.override.yml
# Ce fichier est automatiquement chargé et override docker-compose.yml
# Utiliser pour les configurations locales spécifiques à chaque développeur

version: '3.9'

services:
  api:
    # Override pour MacOS avec performances améliorées
    volumes:
      - ./backend/src:/app/src:cached  # cached plutôt que delegated sur Mac

    # Variables d'environnement additionnelles
    environment:
      DEVELOPER_NAME: ${USER}
      LOCAL_TIMEZONE: ${TZ:-UTC}

  mongodb:
    # Utiliser un port différent localement si 27017 est occupé
    ports:
      - "${MONGO_LOCAL_PORT:-27017}:27017"
```

### Script de Configuration Initiale

```bash
#!/bin/bash
# scripts/setup.sh - Configuration initiale de l'environnement de développement

set -euo pipefail

echo "🚀 Configuration de l'environnement de développement"

# Vérifier Docker
if ! command -v docker &> /dev/null; then
    echo "❌ Docker n'est pas installé"
    exit 1
fi

if ! docker info &> /dev/null; then
    echo "❌ Docker daemon n'est pas démarré"
    exit 1
fi

# Créer le fichier .env si il n'existe pas
if [ ! -f .env ]; then
    echo "📝 Création du fichier .env"
    cp .env.example .env
    echo "⚠️  Pensez à configurer les variables dans .env"
fi

# Créer les dossiers nécessaires
echo "📁 Création des dossiers"
mkdir -p data/mongo
mkdir -p data/redis
mkdir -p logs
mkdir -p backups

# Pull des images
echo "📥 Téléchargement des images Docker"
docker-compose pull

# Build des images custom
echo "🔨 Build des images"
docker-compose build

# Démarrer MongoDB pour l'initialisation
echo "🚀 Démarrage de MongoDB"
docker-compose up -d mongodb

# Attendre que MongoDB soit prêt
echo "⏳ Attente du démarrage de MongoDB..."
until docker-compose exec -T mongodb mongosh --eval "db.adminCommand('ping')" &> /dev/null; do
    sleep 2
done

echo "✅ MongoDB est prêt"

# Charger les seed data
echo "🌱 Chargement des données de test"
make seed

# Démarrer tous les services
echo "🚀 Démarrage de tous les services"
docker-compose up -d

# Afficher l'état
echo ""
echo "✨ Configuration terminée!"
echo ""
echo "📊 Services disponibles:"
echo "  - MongoDB: mongodb://localhost:27017"
echo "  - Mongo Express: http://localhost:8081"
echo "  - API: http://localhost:3000"
echo "  - Frontend: http://localhost:3001"
echo "  - MailHog: http://localhost:8025"
echo ""
echo "📝 Commandes utiles:"
echo "  - make start      : Démarrer l'environnement"
echo "  - make stop       : Arrêter l'environnement"
echo "  - make logs       : Voir les logs"
echo "  - make shell      : Shell MongoDB"
echo "  - make help       : Voir toutes les commandes"
```

---

## Debugging et Profiling

### Configuration VSCode pour Debugging

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker: Attach to Node",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}/backend/src",
      "remoteRoot": "/app/src",
      "protocol": "inspector",
      "restart": true,
      "skipFiles": [
        "<node_internals>/**"
      ]
    },
    {
      "name": "Docker: Run Tests",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "docker-compose",
      "runtimeArgs": [
        "exec",
        "-T",
        "api",
        "npm",
        "test"
      ],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    }
  ]
}
```

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Docker: Start Dev Environment",
      "type": "shell",
      "command": "make start",
      "problemMatcher": [],
      "group": {
        "kind": "build",
        "isDefault": true
      }
    },
    {
      "label": "Docker: Stop Dev Environment",
      "type": "shell",
      "command": "make stop",
      "problemMatcher": []
    },
    {
      "label": "Docker: View Logs",
      "type": "shell",
      "command": "make logs",
      "problemMatcher": [],
      "isBackground": true
    },
    {
      "label": "Docker: MongoDB Shell",
      "type": "shell",
      "command": "make shell",
      "problemMatcher": [],
      "presentation": {
        "reveal": "always",
        "panel": "new"
      }
    },
    {
      "label": "Docker: Restart Services",
      "type": "shell",
      "command": "make restart",
      "problemMatcher": []
    }
  ]
}
```

### MongoDB Profiler en Développement

```javascript
// scripts/setup-profiler.js
// Script pour configurer le profiler MongoDB

const { MongoClient } = require('mongodb');

async function setupProfiler() {
  const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017';
  const client = new MongoClient(uri);

  try {
    await client.connect();
    console.log('📊 Configuration du MongoDB Profiler...');

    const db = client.db('dev_db');

    // Activer le profiler pour toutes les opérations
    await db.command({
      profile: 2, // 0=off, 1=slow ops, 2=all ops
      slowms: 50  // Considérer comme "slow" si > 50ms
    });

    console.log('✅ Profiler activé');

    // Créer des index sur system.profile pour de meilleures performances
    const profileCollection = db.collection('system.profile');
    await profileCollection.createIndex({ ts: -1 });
    await profileCollection.createIndex({ millis: -1 });

    // Fonction helper pour analyser les slow queries
    console.log('\n📈 Top 10 requêtes les plus lentes:');
    const slowQueries = await profileCollection
      .find()
      .sort({ millis: -1 })
      .limit(10)
      .toArray();

    slowQueries.forEach((query, index) => {
      console.log(`\n${index + 1}. ${query.op} on ${query.ns}`);
      console.log(`   Durée: ${query.millis}ms`);
      console.log(`   Command: ${JSON.stringify(query.command, null, 2)}`);
    });

  } catch (error) {
    console.error('❌ Erreur:', error);
  } finally {
    await client.close();
  }
}

setupProfiler();
```

```bash
#!/bin/bash
# scripts/analyze-queries.sh
# Analyse les slow queries MongoDB

echo "🔍 Analyse des slow queries..."

docker-compose exec -T mongodb mongosh dev_db --eval "
  db.system.profile.find()
    .sort({ millis: -1 })
    .limit(10)
    .forEach(doc => {
      print('\\n' + '='.repeat(80));
      print('Operation: ' + doc.op + ' on ' + doc.ns);
      print('Duration: ' + doc.millis + 'ms');
      print('Timestamp: ' + doc.ts);
      print('Command: ' + JSON.stringify(doc.command, null, 2));

      if (doc.planSummary) {
        print('Plan: ' + doc.planSummary);
      }

      if (doc.docsExamined) {
        print('Docs Examined: ' + doc.docsExamined);
      }

      if (doc.keysExamined) {
        print('Keys Examined: ' + doc.keysExamined);
      }
    })
"
```

---

## Tests et Intégration Continue

### Docker Compose pour Tests

```yaml
# docker-compose.test.yml
version: '3.9'

services:
  mongodb-test:
    image: mongo:7.0.5
    container_name: test-mongodb

    tmpfs:
      - /data/db  # En mémoire pour tests rapides

    command:
      - --storageEngine=wiredTiger
      - --wiredTigerCacheSizeGB=0.25
      - --noauth
      - --quiet

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      timeout: 3s
      retries: 5

    networks:
      - test_network

  tests:
    build:
      context: ./backend
      dockerfile: Dockerfile.test

    container_name: test-runner

    environment:
      NODE_ENV: test
      MONGODB_URI: mongodb://mongodb-test:27017/test_db
      CI: true

    volumes:
      - ./backend/src:/app/src:ro
      - ./backend/tests:/app/tests:ro
      - test_coverage:/app/coverage

    command: npm run test:ci

    depends_on:
      mongodb-test:
        condition: service_healthy

    networks:
      - test_network

volumes:
  test_coverage:

networks:
  test_network:
    driver: bridge
```

```dockerfile
# backend/Dockerfile.test
FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && \
    npm ci --only=development

# Copy source
COPY . .

# Run tests
CMD ["npm", "run", "test:ci"]
```

```json
// backend/package.json (scripts de test)
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watchAll",
    "test:ci": "jest --ci --coverage --maxWorkers=2",
    "test:integration": "jest --testPathPattern=integration",
    "test:unit": "jest --testPathPattern=unit"
  },
  "jest": {
    "testEnvironment": "node",
    "coverageDirectory": "coverage",
    "collectCoverageFrom": [
      "src/**/*.js",
      "!src/**/*.test.js"
    ],
    "testMatch": [
      "**/tests/**/*.test.js"
    ]
  }
}
```

### Tests d'Intégration MongoDB

```javascript
// backend/tests/integration/mongodb.test.js
const { MongoClient } = require('mongodb');

describe('MongoDB Integration Tests', () => {
  let client;
  let db;

  beforeAll(async () => {
    const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017';
    client = new MongoClient(uri);
    await client.connect();
    db = client.db('test_db');
  });

  afterAll(async () => {
    await client.close();
  });

  beforeEach(async () => {
    // Clean database before each test
    const collections = await db.listCollections().toArray();
    for (const collection of collections) {
      await db.collection(collection.name).deleteMany({});
    }
  });

  describe('CRUD Operations', () => {
    test('should insert and retrieve a document', async () => {
      const collection = db.collection('users');

      const user = {
        email: 'test@example.com',
        name: 'Test User',
        createdAt: new Date()
      };

      const result = await collection.insertOne(user);
      expect(result.insertedId).toBeDefined();

      const retrieved = await collection.findOne({ _id: result.insertedId });
      expect(retrieved.email).toBe(user.email);
      expect(retrieved.name).toBe(user.name);
    });

    test('should update a document', async () => {
      const collection = db.collection('users');

      const user = await collection.insertOne({
        email: 'test@example.com',
        name: 'Old Name'
      });

      await collection.updateOne(
        { _id: user.insertedId },
        { $set: { name: 'New Name' } }
      );

      const updated = await collection.findOne({ _id: user.insertedId });
      expect(updated.name).toBe('New Name');
    });

    test('should delete a document', async () => {
      const collection = db.collection('users');

      const user = await collection.insertOne({
        email: 'test@example.com',
        name: 'Test User'
      });

      await collection.deleteOne({ _id: user.insertedId });

      const deleted = await collection.findOne({ _id: user.insertedId });
      expect(deleted).toBeNull();
    });
  });

  describe('Indexes', () => {
    test('should create and use indexes', async () => {
      const collection = db.collection('users');

      // Create index
      await collection.createIndex({ email: 1 }, { unique: true });

      // Insert document
      await collection.insertOne({
        email: 'test@example.com',
        name: 'Test User'
      });

      // Verify index is used
      const explain = await collection
        .find({ email: 'test@example.com' })
        .explain();

      expect(explain.executionStats.executionSuccess).toBe(true);
      expect(explain.queryPlanner.winningPlan.inputStage.indexName).toBe('email_1');
    });

    test('should enforce unique constraint', async () => {
      const collection = db.collection('users');

      await collection.createIndex({ email: 1 }, { unique: true });

      await collection.insertOne({
        email: 'test@example.com',
        name: 'User 1'
      });

      // Should throw duplicate key error
      await expect(
        collection.insertOne({
          email: 'test@example.com',
          name: 'User 2'
        })
      ).rejects.toThrow(/duplicate key/);
    });
  });

  describe('Transactions', () => {
    test('should commit transaction', async () => {
      const session = client.startSession();

      try {
        await session.withTransaction(async () => {
          const users = db.collection('users');
          const orders = db.collection('orders');

          const user = await users.insertOne(
            { email: 'test@example.com' },
            { session }
          );

          await orders.insertOne(
            { userId: user.insertedId, amount: 100 },
            { session }
          );
        });

        // Verify data is committed
        const userCount = await db.collection('users').countDocuments();
        const orderCount = await db.collection('orders').countDocuments();

        expect(userCount).toBe(1);
        expect(orderCount).toBe(1);

      } finally {
        await session.endSession();
      }
    });

    test('should rollback transaction on error', async () => {
      const session = client.startSession();

      try {
        await session.withTransaction(async () => {
          const users = db.collection('users');

          await users.insertOne(
            { email: 'test@example.com' },
            { session }
          );

          // Simulate error
          throw new Error('Transaction error');
        });
      } catch (error) {
        expect(error.message).toBe('Transaction error');
      } finally {
        await session.endSession();
      }

      // Verify no data is committed
      const userCount = await db.collection('users').countDocuments();
      expect(userCount).toBe(0);
    });
  });
});
```

---

## Optimisations Performance pour Développement

### Volume Performance sur macOS et Windows

```yaml
# docker-compose.yml avec optimisations volumes

services:
  api:
    volumes:
      # macOS: utiliser 'cached' pour de meilleures performances
      - ./backend/src:/app/src:cached

      # Alternative: utiliser docker-sync (nécessite configuration séparée)
      # - api-sync:/app/src:nocopy

      # Anonymous volume pour node_modules (performance)
      - /app/node_modules

# Configuration docker-sync (optionnelle, pour macOS)
# docker-sync.yml
# version: "2"
# syncs:
#   api-sync:
#     src: './backend/src'
#     sync_strategy: 'native_osx'
#     sync_excludes: ['node_modules', '.git']
```

### MongoDB avec tmpfs pour Tests

```yaml
services:
  mongodb-test:
    image: mongo:7.0.5

    # Utiliser tmpfs pour des tests ultra-rapides
    tmpfs:
      - /data/db:size=512M,mode=1777
      - /data/configdb:size=10M,mode=1777

    # Configuration minimale
    command:
      - --storageEngine=wiredTiger
      - --wiredTigerCacheSizeGB=0.25
      - --nojournal  # Pas de journal pour les tests
      - --quiet
```

### Script de Warm-up

```bash
#!/bin/bash
# scripts/warmup.sh
# Pre-charge les données et prépare l'environnement

echo "🔥 Warm-up de l'environnement de développement..."

# Attendre que tous les services soient prêts
docker-compose up -d
sleep 10

# Créer les index
echo "📊 Création des index..."
docker-compose exec -T mongodb mongosh dev_db --eval "
  db.users.createIndex({ email: 1 }, { unique: true });
  db.products.createIndex({ name: 'text', description: 'text' });
  db.orders.createIndex({ userId: 1, createdAt: -1 });
"

# Pré-charger le cache
echo "💾 Pré-chargement du cache..."
curl -s http://localhost:3000/api/health > /dev/null

# Générer des données de test volumineuses si nécessaire
if [ "${GENERATE_LARGE_DATASET:-false}" = "true" ]; then
  echo "🌱 Génération d'un large jeu de données..."
  docker-compose exec -T mongodb mongosh dev_db < ./scripts/generate-large-dataset.js
fi

echo "✅ Warm-up terminé!"
```

---

## Troubleshooting et Diagnostics

### Health Check Dashboard

```bash
#!/bin/bash
# scripts/health.sh
# Affiche l'état de santé de tous les services

echo "🏥 Health Check Dashboard"
echo "========================"
echo ""

# MongoDB
echo "📊 MongoDB:"
if docker-compose exec -T mongodb mongosh --eval "db.adminCommand('ping')" &> /dev/null; then
  echo "  ✅ Status: Running"
  MONGO_VERSION=$(docker-compose exec -T mongodb mongosh --quiet --eval "db.version()")
  echo "  📌 Version: $MONGO_VERSION"

  MONGO_STATS=$(docker-compose exec -T mongodb mongosh dev_db --quiet --eval "
    const stats = db.stats();
    print('Collections: ' + stats.collections);
    print('Documents: ' + Object.values(db.getCollectionNames().map(name => db[name].countDocuments())).reduce((a,b) => a+b, 0));
    print('Size: ' + (stats.dataSize / 1024 / 1024).toFixed(2) + ' MB');
  ")
  echo "$MONGO_STATS" | sed 's/^/  /'
else
  echo "  ❌ Status: Down"
fi

echo ""

# API
echo "🚀 API:"
if curl -sf http://localhost:3000/health &> /dev/null; then
  echo "  ✅ Status: Running"
  API_VERSION=$(curl -s http://localhost:3000/health | jq -r '.version // "unknown"')
  echo "  📌 Version: $API_VERSION"
else
  echo "  ❌ Status: Down or not responding"
fi

echo ""

# Frontend
echo "🎨 Frontend:"
if curl -sf http://localhost:3001 &> /dev/null; then
  echo "  ✅ Status: Running"
else
  echo "  ❌ Status: Down or not responding"
fi

echo ""

# Redis
echo "💾 Redis:"
if docker-compose exec -T redis redis-cli ping &> /dev/null; then
  echo "  ✅ Status: Running"
  REDIS_INFO=$(docker-compose exec -T redis redis-cli info | grep -E "redis_version|used_memory_human")
  echo "$REDIS_INFO" | sed 's/^/  /'
else
  echo "  ❌ Status: Down"
fi

echo ""

# Docker resources
echo "🐳 Docker Resources:"
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" | grep -E "dev-|CONTAINER"
```

### Log Aggregation Script

```bash
#!/bin/bash
# scripts/logs-all.sh
# Agrège et formate les logs de tous les services

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Fonction pour formater les logs
format_log() {
  local service=$1
  local color=$2

  while IFS= read -r line; do
    timestamp=$(date +"%H:%M:%S")
    echo -e "${color}[$timestamp]${NC} ${BLUE}[$service]${NC} $line"
  done
}

# Suivre les logs de plusieurs services
{
  docker-compose logs -f mongodb 2>&1 | format_log "MongoDB" "$GREEN" &
  docker-compose logs -f api 2>&1 | format_log "API" "$YELLOW" &
  docker-compose logs -f frontend 2>&1 | format_log "Frontend" "$BLUE" &
  docker-compose logs -f redis 2>&1 | format_log "Redis" "$RED" &

  wait
}
```

### Reset Script Complet

```bash
#!/bin/bash
# scripts/reset.sh
# Reset complet de l'environnement

set -euo pipefail

echo "⚠️  ATTENTION: Cette opération va supprimer toutes les données!"
read -p "Êtes-vous sûr? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
  echo "❌ Opération annulée"
  exit 0
fi

echo "🧹 Nettoyage en cours..."

# Arrêter tous les conteneurs
echo "⏹  Arrêt des conteneurs..."
docker-compose down -v

# Supprimer les volumes
echo "🗑️  Suppression des volumes..."
docker volume rm $(docker volume ls -q | grep dev) 2>/dev/null || true

# Nettoyer les données locales
echo "🗑️  Suppression des données locales..."
rm -rf data/mongo/*
rm -rf data/redis/*
rm -rf logs/*

# Nettoyer les images non utilisées
echo "🧹 Nettoyage des images Docker..."
docker image prune -f

# Rebuilder
echo "🔨 Rebuild des images..."
docker-compose build --no-cache

# Redémarrer
echo "🚀 Redémarrage de l'environnement..."
make start

# Recharger les seed data
echo "🌱 Rechargement des seed data..."
sleep 10
make seed

echo "✅ Reset terminé!"
echo ""
echo "📊 Nouvel environnement prêt à l'emploi"
```

---

## Best Practices pour le Développement

### Checklist Environnement de Dev

```yaml
# dev-checklist.yml
---
configuration:
  - ✅ .env.example créé et documenté
  - ✅ .env ajouté au .gitignore
  - ✅ docker-compose.yml avec services de base
  - ✅ docker-compose.override.yml pour config locale
  - ✅ Makefile avec commandes courantes

data:
  - ✅ Scripts d'initialisation MongoDB
  - ✅ Seed data pour développement
  - ✅ Volumes persistants configurés
  - ✅ Strategy de backup en place

performance:
  - ✅ Volumes optimisés (cached/delegated)
  - ✅ Anonymous volumes pour node_modules
  - ✅ MongoDB avec cache réduit
  - ✅ Hot reload configuré

debugging:
  - ✅ Ports de debug exposés
  - ✅ Configuration VSCode/IDE
  - ✅ Profiler MongoDB activé
  - ✅ Logs structurés

testing:
  - ✅ docker-compose.test.yml
  - ✅ Tests d'intégration MongoDB
  - ✅ CI/CD local avec Act
  - ✅ Coverage rapports

documentation:
  - ✅ README.md à jour
  - ✅ Architecture documentée
  - ✅ Troubleshooting guide
  - ✅ Onboarding nouveaux devs

tooling:
  - ✅ Mongo Express ou Adminer
  - ✅ MailHog pour emails
  - ✅ Health checks endpoints
  - ✅ Scripts helper (Makefile)
```

### Tips et Astuces

**1. Utiliser des alias pour gagner du temps**
```bash
# ~/.bashrc ou ~/.zshrc
alias dc='docker-compose'
alias dcup='docker-compose up -d'
alias dcdown='docker-compose down'
alias dclogs='docker-compose logs -f'
alias dcps='docker-compose ps'
alias dcshell='docker-compose exec mongodb mongosh'
```

**2. Watcher pour rebuild automatique**
```yaml
services:
  api:
    build:
      context: ./backend
      target: development

    # Rebuild automatique sur changement de Dockerfile
    labels:
      - "dev.rebuild=true"
```

**3. Utiliser ctop pour monitoring en temps réel**
```bash
# Installation
brew install ctop  # macOS
# ou
sudo wget https://github.com/bcicen/ctop/releases/download/0.7.7/ctop-0.7.7-linux-amd64 -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop

# Utilisation
ctop
```

---

## Conclusion

Un environnement de développement Docker bien configuré pour MongoDB permet :
- **Rapidité** : Démarrage en secondes, pas en minutes
- **Isolation** : Chaque projet avec ses dépendances
- **Reproductibilité** : Même environnement pour toute l'équipe
- **Productivité** : Moins de "ça marche sur ma machine"

Les points clés à retenir :
1. Utiliser des volumes nommés pour la persistance
2. Optimiser les volumes pour les performances (cached/delegated)
3. Prévoir des seed data et scripts d'initialisation
4. Configurer le hot reload pour le développement itératif
5. Intégrer les outils de debugging et profiling
6. Automatiser avec un Makefile
7. Documenter pour faciliter l'onboarding

---


⏭️ [Kubernetes et MongoDB](/18-devops-deploiement/04-kubernetes-mongodb.md)
