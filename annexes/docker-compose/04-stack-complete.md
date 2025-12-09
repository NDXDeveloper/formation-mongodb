🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.4 - Stack Complète (MongoDB + Mongo Express + Application)

## Introduction

Configuration Docker Compose pour déployer un **environnement complet de développement** avec MongoDB, interface web de gestion, application exemple et outils complémentaires. Solution clé en main pour démarrer rapidement.

### 🎯 Cas d'usage

- Environnement de développement complet
- Démos et présentations
- Prototypage rapide d'applications
- Formation avec outils visuels
- POC full-stack
- Tests d'intégration end-to-end

### ✅ Composants Inclus

```markdown
✅ MongoDB (Standalone ou Replica Set)
✅ Mongo Express (UI Web)
✅ Application exemple (Node.js/Python/autre)
✅ Reverse Proxy (Nginx - optionnel)
✅ Monitoring (Prometheus/Grafana - optionnel)
✅ Redis (Cache - optionnel)
✅ Volumes persistants
✅ Réseau dédié
```

---

## Architecture

### 🏗️ Stack Standard

```
┌────────────────────────────────────────┐
│           Stack Complète               │
├────────────────────────────────────────┤
│                                        │
│  ┌─────────────────────────────────┐   │
│  │         Nginx (8080)            │   │
│  │      Reverse Proxy              │   │
│  └──────────┬──────────────────────┘   │
│             │                          │
│     ┌───────┴────────┐                 │
│     │                │                 │
│  ┌──▼──────────┐  ┌──▼──────────┐      │
│  │   Web App   │  │Mongo Express│      │
│  │   (3000)    │  │   (8081)    │      │
│  └──────┬──────┘  └──────┬──────┘      │
│         │                │             │
│         └────────┬───────┘             │
│                  │                     │
│         ┌────────▼────────┐            │
│         │    MongoDB      │            │
│         │    (27017)      │            │
│         └─────────────────┘            │
│                                        │
│            app_network                 │
└────────────────────────────────────────┘
```

---

## Stack Minimale (MongoDB + Mongo Express)

### 🚀 Démarrage Rapide

**Structure** :
```
stack-minimal/
├── docker-compose.yml
├── .env
└── .gitignore
```

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: unless-stopped

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_DATABASE}

    volumes:
      - mongodb_data:/data/db

    networks:
      - app_network

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped

    ports:
      - "8081:8081"

    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: ${MONGO_ROOT_USERNAME}
      ME_CONFIG_MONGODB_ADMINPASSWORD: ${MONGO_ROOT_PASSWORD}
      ME_CONFIG_MONGODB_URL: mongodb://${MONGO_ROOT_USERNAME}:${MONGO_ROOT_PASSWORD}@mongodb:27017/
      ME_CONFIG_BASICAUTH_USERNAME: ${ME_USERNAME}
      ME_CONFIG_BASICAUTH_PASSWORD: ${ME_PASSWORD}

    depends_on:
      mongodb:
        condition: service_healthy

    networks:
      - app_network

volumes:
  mongodb_data:

networks:
  app_network:
    driver: bridge
```

**.env** :
```bash
# MongoDB
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=SecurePassword123!
MONGO_DATABASE=myapp

# Mongo Express
ME_USERNAME=admin
ME_PASSWORD=admin123
```

**.gitignore** :
```
.env
data/
logs/
node_modules/
__pycache__/
```

**Utilisation** :
```bash
# Démarrer
docker-compose up -d

# Accès
# MongoDB: mongodb://admin:SecurePassword123!@localhost:27017
# Mongo Express: http://localhost:8081 (admin/admin123)
```

---

## Stack Complète avec Application Node.js

### 📦 Structure Complète

```
stack-nodejs/
├── docker-compose.yml
├── .env
├── .gitignore
├── nginx/
│   └── nginx.conf
├── app/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   │   └── index.js
│   └── .dockerignore
└── init-scripts/
    └── 01-init.js
```

---

### 🐳 docker-compose.yml Complet

```yaml
version: '3.8'

services:
  # ============================================
  # MongoDB
  # ============================================
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: unless-stopped

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_DATABASE}

    volumes:
      - mongodb_data:/data/db
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

    networks:
      - app_network

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s

  # ============================================
  # Mongo Express (Interface Web)
  # ============================================
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped

    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: ${MONGO_ROOT_USERNAME}
      ME_CONFIG_MONGODB_ADMINPASSWORD: ${MONGO_ROOT_PASSWORD}
      ME_CONFIG_MONGODB_URL: mongodb://${MONGO_ROOT_USERNAME}:${MONGO_ROOT_PASSWORD}@mongodb:27017/
      ME_CONFIG_BASICAUTH_USERNAME: ${ME_USERNAME}
      ME_CONFIG_BASICAUTH_PASSWORD: ${ME_PASSWORD}

    depends_on:
      mongodb:
        condition: service_healthy

    networks:
      - app_network

  # ============================================
  # Application Node.js
  # ============================================
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: nodejs-app
    restart: unless-stopped

    environment:
      NODE_ENV: ${NODE_ENV:-development}
      MONGODB_URI: mongodb://${MONGO_ROOT_USERNAME}:${MONGO_ROOT_PASSWORD}@mongodb:27017/${MONGO_DATABASE}?authSource=admin
      PORT: 3000

    volumes:
      - ./app/src:/app/src:ro  # Hot reload en dev

    depends_on:
      mongodb:
        condition: service_healthy

    networks:
      - app_network

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ============================================
  # Nginx (Reverse Proxy)
  # ============================================
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped

    ports:
      - "8080:80"

    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro

    depends_on:
      - app
      - mongo-express

    networks:
      - app_network

volumes:
  mongodb_data:

networks:
  app_network:
    driver: bridge
```

---

### 📝 Application Node.js

**app/Dockerfile** :
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY src ./src

# Health check
RUN apk add --no-cache curl

# Non-root user
USER node

EXPOSE 3000

CMD ["node", "src/index.js"]
```

**app/package.json** :
```json
{
  "name": "mongodb-app",
  "version": "1.0.0",
  "description": "Sample MongoDB Application",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongodb": "^6.3.0",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

**app/src/index.js** :
```javascript
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
const port = process.env.PORT || 3000;
const mongoUri = process.env.MONGODB_URI;

let db;

// Middleware
app.use(express.json());

// Connect to MongoDB
async function connectDB() {
  try {
    const client = new MongoClient(mongoUri);
    await client.connect();
    db = client.db();
    console.log('Connected to MongoDB');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
}

// Routes
app.get('/', (req, res) => {
  res.json({
    message: 'MongoDB Stack API',
    version: '1.0.0',
    endpoints: {
      health: '/health',
      users: '/users',
      stats: '/stats'
    }
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date() });
});

app.get('/users', async (req, res) => {
  try {
    const users = await db.collection('users').find().toArray();
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/users', async (req, res) => {
  try {
    const user = req.body;
    const result = await db.collection('users').insertOne({
      ...user,
      createdAt: new Date()
    });
    res.status(201).json({
      _id: result.insertedId,
      ...user
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/stats', async (req, res) => {
  try {
    const stats = await db.stats();
    const userCount = await db.collection('users').countDocuments();
    res.json({
      database: stats.db,
      collections: stats.collections,
      dataSize: stats.dataSize,
      users: userCount
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Start server
connectDB().then(() => {
  app.listen(port, '0.0.0.0', () => {
    console.log(`App listening on port ${port}`);
  });
});
```

**app/.dockerignore** :
```
node_modules
npm-debug.log
.env
.git
.gitignore
README.md
```

---

### 🌐 Configuration Nginx

**nginx/nginx.conf** :
```nginx
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    upstream mongo-express {
        server mongo-express:8081;
    }

    server {
        listen 80;
        server_name localhost;

        # Application API
        location /api/ {
            proxy_pass http://app/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_cache_bypass $http_upgrade;
        }

        # Mongo Express
        location /mongo/ {
            proxy_pass http://mongo-express/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        # Health check
        location /health {
            proxy_pass http://app/health;
        }

        # Default
        location / {
            return 200 'MongoDB Stack is running\n';
            add_header Content-Type text/plain;
        }
    }
}
```

---

### 📜 Script d'Initialisation

**init-scripts/01-init.js** :
```javascript
// Initialize database
db = db.getSiblingDB('myapp');

// Create users collection with validation
db.createCollection('users', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['name', 'email'],
      properties: {
        name: {
          bsonType: 'string',
          description: 'must be a string and is required'
        },
        email: {
          bsonType: 'string',
          pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
          description: 'must be a valid email and is required'
        },
        age: {
          bsonType: 'int',
          minimum: 0,
          maximum: 150,
          description: 'must be an integer between 0 and 150'
        }
      }
    }
  }
});

// Create indexes
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ name: 1 });

// Insert sample data
db.users.insertMany([
  {
    name: 'John Doe',
    email: 'john@example.com',
    age: 30,
    role: 'developer',
    createdAt: new Date()
  },
  {
    name: 'Jane Smith',
    email: 'jane@example.com',
    age: 28,
    role: 'designer',
    createdAt: new Date()
  },
  {
    name: 'Bob Johnson',
    email: 'bob@example.com',
    age: 35,
    role: 'manager',
    createdAt: new Date()
  }
]);

// Create application user
db.getSiblingDB('admin').createUser({
  user: 'appuser',
  pwd: 'apppassword',
  roles: [
    { role: 'readWrite', db: 'myapp' }
  ]
});

print('Database initialized successfully');
print('Users created: ' + db.users.countDocuments());
```

---

### 🚀 Déploiement et Utilisation

```bash
# 1. Créer le projet
mkdir stack-nodejs && cd stack-nodejs
mkdir -p app/src nginx init-scripts

# 2. Créer tous les fichiers
# (copier docker-compose.yml, app/*, nginx/*, etc.)

# 3. Build et démarrer
docker-compose up -d --build

# 4. Vérifier les services
docker-compose ps

# 5. Voir les logs
docker-compose logs -f app

# 6. Tester l'API
curl http://localhost:8080/api/
curl http://localhost:8080/api/users
curl http://localhost:8080/api/stats

# 7. Créer un utilisateur
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","age":25}'

# 8. Accéder à Mongo Express
# http://localhost:8080/mongo/ (admin/admin123)

# 9. Health check
curl http://localhost:8080/health
```

---

## Stack avec Python (FastAPI)

### 🐍 Application Python

**app-python/Dockerfile** :
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source
COPY src ./src

# Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**app-python/requirements.txt** :
```
fastapi==0.109.0
uvicorn[standard]==0.27.0
pymongo==4.6.1
pydantic==2.5.3
python-dotenv==1.0.0
```

**app-python/src/main.py** :
```python
from fastapi import FastAPI, HTTPException
from pymongo import MongoClient
from pydantic import BaseModel, EmailStr
from typing import Optional
import os
from datetime import datetime

app = FastAPI(title="MongoDB Stack API")

# MongoDB connection
client = MongoClient(os.getenv("MONGODB_URI"))
db = client[os.getenv("MONGO_DATABASE", "myapp")]

# Models
class User(BaseModel):
    name: str
    email: EmailStr
    age: Optional[int] = None
    role: Optional[str] = None

class UserResponse(User):
    id: str
    createdAt: datetime

# Routes
@app.get("/")
def read_root():
    return {
        "message": "MongoDB Stack API",
        "version": "1.0.0",
        "docs": "/docs"
    }

@app.get("/health")
def health_check():
    try:
        client.admin.command('ping')
        return {"status": "healthy", "mongodb": "connected"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))

@app.get("/users")
def get_users():
    users = list(db.users.find())
    for user in users:
        user["id"] = str(user.pop("_id"))
    return users

@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(user: User):
    user_dict = user.model_dump()
    user_dict["createdAt"] = datetime.now()

    result = db.users.insert_one(user_dict)
    user_dict["id"] = str(result.inserted_id)

    return user_dict

@app.get("/stats")
def get_stats():
    return {
        "database": db.name,
        "collections": db.list_collection_names(),
        "users": db.users.count_documents({})
    }
```

**docker-compose.yml** (extrait) :
```yaml
services:
  # ... mongodb, mongo-express ...

  app-python:
    build:
      context: ./app-python
    container_name: python-app
    environment:
      MONGODB_URI: mongodb://admin:password@mongodb:27017/
      MONGO_DATABASE: myapp
    ports:
      - "8000:8000"
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app_network
```

---

## Stack Avancée avec Monitoring

### 📊 Ajout de Prometheus et Grafana

**docker-compose-monitoring.yml** :
```yaml
version: '3.8'

services:
  # ... mongodb, mongo-express, app ...

  # MongoDB Exporter
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb-exporter
    restart: unless-stopped

    command:
      - '--mongodb.uri=mongodb://admin:${MONGO_ROOT_PASSWORD}@mongodb:27017'
      - '--collect-all'

    ports:
      - "9216:9216"

    depends_on:
      - mongodb

    networks:
      - app_network

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped

    ports:
      - "9090:9090"

    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus

    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

    networks:
      - app_network

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped

    ports:
      - "3001:3000"

    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_USER:-admin}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_INSTALL_PLUGINS: grafana-clock-panel

    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro

    depends_on:
      - prometheus

    networks:
      - app_network

volumes:
  mongodb_data:
  prometheus_data:
  grafana_data:

networks:
  app_network:
    driver: bridge
```

**prometheus/prometheus.yml** :
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  - job_name: 'app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: '/metrics'
```

---

## Stack avec Redis (Cache)

### 💾 Ajout de Redis

```yaml
services:
  # ... mongodb, mongo-express, app ...

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped

    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}

    ports:
      - "6379:6379"

    volumes:
      - redis_data:/data

    networks:
      - app_network

    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  redis_data:
```

**Application avec cache** :
```javascript
const redis = require('redis');

const redisClient = redis.createClient({
  url: `redis://:${process.env.REDIS_PASSWORD}@redis:6379`
});

redisClient.connect();

// Endpoint avec cache
app.get('/users', async (req, res) => {
  try {
    // Vérifier le cache
    const cached = await redisClient.get('users');
    if (cached) {
      return res.json(JSON.parse(cached));
    }

    // Query MongoDB
    const users = await db.collection('users').find().toArray();

    // Mettre en cache (5 minutes)
    await redisClient.setEx('users', 300, JSON.stringify(users));

    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

---

## Stack Frontend (React + Backend)

### ⚛️ Configuration Complète

```yaml
services:
  mongodb:
    # ... (config MongoDB)

  backend:
    build: ./backend
    container_name: backend
    environment:
      MONGODB_URI: mongodb://admin:password@mongodb:27017/myapp
      PORT: 5000
    networks:
      - app_network

  frontend:
    build: ./frontend
    container_name: frontend
    environment:
      REACT_APP_API_URL: http://localhost:8080/api
    networks:
      - app_network

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx/frontend.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - app_network
```

**frontend/Dockerfile** :
```dockerfile
FROM node:18-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
```

---

## Commandes de Gestion

### 🛠️ Commandes Courantes

```bash
# Démarrer la stack
docker-compose up -d

# Démarrer avec rebuild
docker-compose up -d --build

# Voir les services
docker-compose ps

# Logs de tous les services
docker-compose logs -f

# Logs d'un service spécifique
docker-compose logs -f app

# Redémarrer un service
docker-compose restart app

# Arrêter la stack
docker-compose stop

# Arrêter et supprimer
docker-compose down

# Arrêter et supprimer avec volumes
docker-compose down -v

# Voir les stats
docker stats

# Exécuter une commande
docker-compose exec app sh
docker-compose exec mongodb mongosh

# Rebuild d'un service
docker-compose build app
docker-compose up -d app
```

---

### 📊 Scripts Utilitaires

**scripts/reset.sh** :
```bash
#!/bin/bash
# Reset complet de la stack

echo "Stopping all services..."
docker-compose down -v

echo "Removing old images..."
docker-compose down --rmi local

echo "Rebuilding..."
docker-compose build --no-cache

echo "Starting fresh..."
docker-compose up -d

echo "Waiting for services..."
sleep 30

echo "Services status:"
docker-compose ps

echo "Reset completed!"
```

**scripts/backup.sh** :
```bash
#!/bin/bash
# Backup MongoDB

DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="./backups"

mkdir -p $BACKUP_DIR

echo "Backing up MongoDB..."
docker-compose exec -T mongodb mongodump \
  --username=admin \
  --password=$MONGO_ROOT_PASSWORD \
  --authenticationDatabase=admin \
  --gzip \
  --archive > $BACKUP_DIR/backup-$DATE.gz

echo "Backup completed: $BACKUP_DIR/backup-$DATE.gz"
```

**scripts/test-api.sh** :
```bash
#!/bin/bash
# Test de l'API

BASE_URL="http://localhost:8080/api"

echo "=== Testing API ==="

# Health check
echo -e "\n1. Health Check"
curl -s $BASE_URL/health | jq

# Get users
echo -e "\n2. Get Users"
curl -s $BASE_URL/users | jq

# Create user
echo -e "\n3. Create User"
curl -s -X POST $BASE_URL/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","age":25}' | jq

# Stats
echo -e "\n4. Stats"
curl -s $BASE_URL/stats | jq
```

---

## Monitoring et Health Checks

### 🏥 Health Checks Complets

**scripts/health-check.sh** :
```bash
#!/bin/bash
# Vérifier la santé de tous les services

echo "========================================="
echo "Stack Health Check"
echo "========================================="

# MongoDB
echo -e "\n### MongoDB ###"
docker-compose exec -T mongodb mongosh \
  --quiet --eval "db.adminCommand('ping')" \
  && echo "✓ MongoDB is healthy" \
  || echo "✗ MongoDB is down"

# Mongo Express
echo -e "\n### Mongo Express ###"
curl -sf http://localhost:8081 > /dev/null \
  && echo "✓ Mongo Express is healthy" \
  || echo "✗ Mongo Express is down"

# Application
echo -e "\n### Application ###"
curl -sf http://localhost:8080/api/health > /dev/null \
  && echo "✓ Application is healthy" \
  || echo "✗ Application is down"

# Nginx
echo -e "\n### Nginx ###"
curl -sf http://localhost:8080 > /dev/null \
  && echo "✓ Nginx is healthy" \
  || echo "✗ Nginx is down"

# Docker containers
echo -e "\n### Containers Status ###"
docker-compose ps

echo -e "\n========================================="
```

---

## Tests d'Intégration

### 🧪 Tests End-to-End

**tests/integration.test.js** :
```javascript
const axios = require('axios');
const { expect } = require('chai');

const API_URL = 'http://localhost:8080/api';

describe('Integration Tests', () => {
  it('should connect to API', async () => {
    const response = await axios.get(`${API_URL}/`);
    expect(response.status).to.equal(200);
  });

  it('should check health', async () => {
    const response = await axios.get(`${API_URL}/health`);
    expect(response.status).to.equal(200);
    expect(response.data.status).to.equal('healthy');
  });

  it('should list users', async () => {
    const response = await axios.get(`${API_URL}/users`);
    expect(response.status).to.equal(200);
    expect(response.data).to.be.an('array');
  });

  it('should create a user', async () => {
    const newUser = {
      name: 'Integration Test User',
      email: `test-${Date.now()}@example.com`,
      age: 30
    };

    const response = await axios.post(`${API_URL}/users`, newUser);
    expect(response.status).to.equal(201);
    expect(response.data.name).to.equal(newUser.name);
  });

  it('should get stats', async () => {
    const response = await axios.get(`${API_URL}/stats`);
    expect(response.status).to.equal(200);
    expect(response.data).to.have.property('users');
  });
});
```

**Exécution** :
```bash
# Démarrer la stack
docker-compose up -d

# Attendre que tout soit prêt
sleep 30

# Lancer les tests
npm test

# Ou avec Docker
docker run --rm --network stack_app_network \
  -v $(pwd)/tests:/tests \
  node:18-alpine \
  sh -c "cd /tests && npm install && npm test"
```

---

## Sécurité

### 🔒 Bonnes Pratiques

**Sécuriser les services** :
```yaml
services:
  mongodb:
    # Ne pas exposer le port en production
    # ports:
    #   - "27017:27017"

    # Bind sur localhost uniquement
    ports:
      - "127.0.0.1:27017:27017"

  mongo-express:
    # Seulement localhost
    ports:
      - "127.0.0.1:8081:8081"

    # Ou utiliser Nginx avec auth
    # Ne pas exposer directement
```

**Nginx avec authentification** :
```nginx
# Dans nginx.conf
location /mongo/ {
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://mongo-express/;
}
```

**Générer .htpasswd** :
```bash
# Créer le fichier
htpasswd -c .htpasswd admin

# Ajouter dans docker-compose.yml
nginx:
  volumes:
    - ./nginx/.htpasswd:/etc/nginx/.htpasswd:ro
```

---

### 🛡️ Secrets Management

**Avec Docker Secrets** :
```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    environment:
      MONGO_INITDB_ROOT_PASSWORD_FILE: /run/secrets/mongo_root_password
    secrets:
      - mongo_root_password

secrets:
  mongo_root_password:
    file: ./secrets/mongo_root_password.txt
```

---

## Production-Ready Stack

### 🏭 Configuration Production

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    restart: always
    ports:
      - "127.0.0.1:27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    volumes:
      - mongodb_data:/data/db
      - ./mongod.conf:/etc/mongod.conf:ro
    command: ["mongod", "--config", "/etc/mongod.conf", "--auth"]
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"
    networks:
      - app_network

  app:
    build:
      context: ./app
      target: production
    restart: always
    environment:
      NODE_ENV: production
      MONGODB_URI: ${MONGODB_URI}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app_network

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - app_network
```

---

## Checklist de Déploiement

### ✅ Avant le Démarrage

```markdown
□ Docker et Docker Compose installés
□ Ressources suffisantes (4 GB RAM minimum)
□ Ports disponibles (8080, 8081, 27017, etc.)
□ .env créé avec mots de passe forts
□ .env dans .gitignore
□ Fichiers de configuration créés
□ Application testée localement
□ Scripts d'initialisation préparés
```

### ✅ Après le Démarrage

```markdown
□ Tous les conteneurs UP : docker-compose ps
□ Healthchecks PASSING
□ MongoDB accessible
□ Mongo Express accessible et fonctionnel
□ Application API répond
□ Nginx route correctement
□ Données initiales présentes
□ Tests d'intégration passent
□ Logs propres (pas d'erreurs)
□ Backup testé
```

---

## Ressources

### 📚 Documentation
- [Docker Compose](https://docs.docker.com/compose/)
- [MongoDB Docker Hub](https://hub.docker.com/_/mongo)
- [Mongo Express](https://github.com/mongo-express/mongo-express)
- [Nginx](https://nginx.org/en/docs/)

### 🔗 Exemples
- [Docker Samples](https://github.com/docker/awesome-compose)
- [MongoDB Examples](https://github.com/mongodb/mongo-docker)

---

## Conclusion

Cette stack complète est **idéale pour** :
- ✅ Développement full-stack rapide
- ✅ Démos et présentations
- ✅ Prototypage avec outils visuels
- ✅ Formation complète
- ✅ Tests d'intégration
- ✅ Environnements de staging

**Adapter pour production** :
- Utiliser des secrets management
- Configurer TLS/SSL
- Implémenter monitoring avancé
- Mettre en place backups automatisés
- Utiliser orchestration (Kubernetes)
- Sécuriser tous les endpoints

**Formation complète** : Retour à [Docker et Docker Compose - README](README.md)

---

**Version** : 1.0
**Testé avec** : MongoDB 7.0, Docker 24.0, Docker Compose 2.23, Node.js 18, Python 3.11

⏭️ [Scripts et Automatisation](/annexes/scripts-automatisation/README.md)
