🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.3 - Configuration Développement Local

## Présentation

### Objectif
Configuration MongoDB légère et rapide pour le développement local, optimisée pour la productivité et le debug, sans les contraintes de production.

### Caractéristiques

- **Démarrage rapide** : Opérationnel en 2 minutes
- **Ressources minimales** : Fonctionne sur laptop
- **Sans sécurité** : Pas d'authentification pour simplifier le dev
- **Logs verbeux** : Facilite le debugging
- **Données éphémères** : Réinitialisation facile
- **Outils intégrés** : Compass, mongosh, Mongo Express

### ⚠️ Important

Cette configuration est **UNIQUEMENT pour le développement local**. Ne JAMAIS l'utiliser en production ou sur un réseau accessible publiquement.

---

## Options de déploiement

### Comparaison des méthodes

| Méthode | Avantages | Inconvénients | Recommandé pour |
|---------|-----------|---------------|-----------------|
| **Standalone local** | Simple, léger | Installation système | Développement simple |
| **Docker** | Isolation, reproductible | Nécessite Docker | Projets modernes |
| **Docker Compose** | Multi-services, orchestration | Plus complexe | Projets avec stack complète |
| **MongoDB Atlas M0** | Gratuit, cloud, zéro config | Latence, limites | Prototypage rapide |

---

## Méthode 1 : Standalone Local

### Prérequis

- **OS** : Windows 10+, macOS 11+, Ubuntu 20.04+
- **RAM** : 4 GB minimum
- **Espace disque** : 5 GB
- **MongoDB** : Version 6.0+, 7.0+ ou 8.0+

### Installation rapide

#### Ubuntu/Debian

```bash
# Importer la clé GPG
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -

# Ajouter le repository
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Installer
sudo apt-get update
sudo apt-get install -y mongodb-org

# Démarrer
sudo systemctl start mongod
sudo systemctl enable mongod
```

#### macOS (Homebrew)

```bash
# Installer via Homebrew
brew tap mongodb/brew
brew install mongodb-community@7.0

# Démarrer
brew services start mongodb-community@7.0

# Vérifier
mongosh
```

#### Windows

```powershell
# Télécharger depuis https://www.mongodb.com/try/download/community
# Installer avec l'assistant
# MongoDB démarre automatiquement comme service Windows

# Tester la connexion
mongosh
```

### Configuration minimale

#### Fichier mongod.conf (développement)

```yaml
# /etc/mongod.conf (Linux/macOS) ou C:\Program Files\MongoDB\Server\7.0\bin\mongod.cfg (Windows)

# Stockage
storage:
  dbPath: /var/lib/mongodb  # Linux/macOS
  # dbPath: C:\data\db      # Windows
  journal:
    enabled: true

# Logs
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log  # Linux/macOS
  # path: C:\data\log\mongod.log     # Windows
  logAppend: true
  verbosity: 1  # Plus verbeux pour le dev

# Réseau
net:
  port: 27017
  bindIp: 127.0.0.1  # Localhost uniquement pour sécurité

# Sécurité - DÉSACTIVÉE pour le dev
# security:
#   authorization: disabled

# Performance dev (moins de cache pour économiser RAM)
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1  # 1 GB seulement
```

### Démarrage manuel (sans service)

```bash
# Linux/macOS
mongod --dbpath ~/mongodb-data --logpath ~/mongodb-data/mongod.log --fork

# Windows (PowerShell)
mongod --dbpath C:\mongodb-data --logpath C:\mongodb-data\mongod.log
```

### Connexion

```bash
# Shell MongoDB
mongosh

# Avec base de données spécifique
mongosh mongodb://localhost:27017/myapp

# Test rapide
mongosh --eval "db.version()"
```

---

## Méthode 2 : Docker (Recommandé)

### Avantages Docker pour le dev

- ✅ Isolation complète du système
- ✅ Version MongoDB facilement changeable
- ✅ Destruction/recréation en secondes
- ✅ Partageable entre l'équipe
- ✅ Identique en CI/CD

### Installation Docker

```bash
# Ubuntu
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# macOS : Télécharger Docker Desktop
# Windows : Télécharger Docker Desktop
```

### Démarrage rapide

```bash
# Lancer MongoDB 7.0
docker run -d \
  --name mongodb-dev \
  -p 27017:27017 \
  -v mongodb-data:/data/db \
  mongo:7.0

# Vérifier
docker ps

# Se connecter
mongosh
```

### Configuration avec variables d'environnement

```bash
# Avec authentification (optionnel pour dev)
docker run -d \
  --name mongodb-dev \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  -v mongodb-data:/data/db \
  mongo:7.0

# Se connecter
mongosh "mongodb://admin:password@localhost:27017"
```

### Commandes Docker utiles

```bash
# Arrêter
docker stop mongodb-dev

# Démarrer
docker start mongodb-dev

# Logs
docker logs -f mongodb-dev

# Shell interactif dans le conteneur
docker exec -it mongodb-dev mongosh

# Détruire (perd les données si pas de volume)
docker rm -f mongodb-dev

# Détruire avec les données
docker rm -f mongodb-dev
docker volume rm mongodb-data
```

---

## Méthode 3 : Docker Compose (Stack Complète)

### Docker Compose : MongoDB + Mongo Express

#### Fichier docker-compose.yml

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb-dev
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      # Pas d'auth pour dev simple
      MONGO_INITDB_DATABASE: myapp
    volumes:
      - mongodb-data:/data/db
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - mongodb-network

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express-dev
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_URL: mongodb://mongodb:27017/
      ME_CONFIG_BASICAUTH: false  # Pas d'auth HTTP pour dev
    depends_on:
      - mongodb
    networks:
      - mongodb-network

volumes:
  mongodb-data:
    driver: local

networks:
  mongodb-network:
    driver: bridge
```

#### Démarrage

```bash
# Créer le fichier docker-compose.yml
# Puis démarrer
docker-compose up -d

# Vérifier
docker-compose ps

# Logs
docker-compose logs -f

# Arrêter
docker-compose down

# Arrêter et supprimer les données
docker-compose down -v
```

#### Accès

- **MongoDB** : `mongodb://localhost:27017`
- **Mongo Express** : `http://localhost:8081`

### Docker Compose : Stack avec Replica Set (dev avancé)

#### Fichier docker-compose-replicaset.yml

```yaml
version: '3.8'

services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    command: ["--replSet", "rs-dev", "--bind_ip_all", "--port", "27017"]
    ports:
      - "27017:27017"
    volumes:
      - mongo1-data:/data/db
    networks:
      - mongo-dev-network
    healthcheck:
      test: echo "try { rs.status() } catch (err) { rs.initiate({_id:'rs-dev',members:[{_id:0,host:'mongo1:27017'},{_id:1,host:'mongo2:27017'},{_id:2,host:'mongo3:27017'}]}) }" | mongosh --port 27017 --quiet
      interval: 10s
      timeout: 10s
      retries: 5
      start_period: 40s

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    command: ["--replSet", "rs-dev", "--bind_ip_all", "--port", "27017"]
    ports:
      - "27018:27017"
    volumes:
      - mongo2-data:/data/db
    networks:
      - mongo-dev-network

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    command: ["--replSet", "rs-dev", "--bind_ip_all", "--port", "27017"]
    ports:
      - "27019:27017"
    volumes:
      - mongo3-data:/data/db
    networks:
      - mongo-dev-network

volumes:
  mongo1-data:
  mongo2-data:
  mongo3-data:

networks:
  mongo-dev-network:
    driver: bridge
```

#### Utilisation

```bash
# Démarrer
docker-compose -f docker-compose-replicaset.yml up -d

# Attendre 30 secondes pour l'initialisation

# Se connecter
mongosh "mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs-dev"

# Vérifier le Replica Set
mongosh --eval "rs.status()"
```

---

## Méthode 4 : MongoDB Atlas (M0 Gratuit)

### Avantages Atlas pour le dev

- ✅ Gratuit (tier M0)
- ✅ Zéro configuration
- ✅ Accessible de partout
- ✅ Backups automatiques
- ✅ Monitoring inclus

### Limitations M0

- 512 MB de stockage
- Connexions limitées
- Pas de Replica Set visible
- Sommeil après inactivité

### Configuration rapide

1. **Créer un compte** : https://www.mongodb.com/cloud/atlas/register
2. **Créer un cluster M0** : Sélectionner "FREE" tier
3. **Whitelist IP** : Ajouter `0.0.0.0/0` (développement)
4. **Créer un utilisateur** : Par exemple `dev` / `dev123`
5. **Récupérer connection string** :

```
mongodb+srv://dev:dev123@cluster0.xxxxx.mongodb.net/myapp?retryWrites=true&w=majority
```

### Connexion

```bash
# mongosh
mongosh "mongodb+srv://dev:dev123@cluster0.xxxxx.mongodb.net/myapp"

# Depuis le code
const uri = "mongodb+srv://dev:dev123@cluster0.xxxxx.mongodb.net/myapp?retryWrites=true&w=majority";
```

---

## Scripts d'initialisation

### Script de seed des données de test

#### init-data.js

```javascript
// init-data.js - Script pour peupler la base de données de dev

// Connexion
db = connect("mongodb://localhost:27017/myapp");

// Supprimer les données existantes
db.users.deleteMany({});
db.products.deleteMany({});
db.orders.deleteMany({});

// Insérer des utilisateurs de test
db.users.insertMany([
  {
    _id: 1,
    username: "john_doe",
    email: "john@example.com",
    firstName: "John",
    lastName: "Doe",
    createdAt: new Date()
  },
  {
    _id: 2,
    username: "jane_smith",
    email: "jane@example.com",
    firstName: "Jane",
    lastName: "Smith",
    createdAt: new Date()
  }
]);

// Insérer des produits de test
db.products.insertMany([
  {
    _id: 1001,
    name: "Laptop Pro 15",
    category: "Electronics",
    price: 1299.99,
    stock: 50,
    tags: ["laptop", "computer", "professional"]
  },
  {
    _id: 1002,
    name: "Wireless Mouse",
    category: "Accessories",
    price: 29.99,
    stock: 200,
    tags: ["mouse", "wireless", "accessory"]
  },
  {
    _id: 1003,
    name: "USB-C Cable",
    category: "Accessories",
    price: 12.99,
    stock: 500,
    tags: ["cable", "usb-c"]
  }
]);

// Insérer des commandes de test
db.orders.insertMany([
  {
    _id: 5001,
    userId: 1,
    items: [
      { productId: 1001, quantity: 1, price: 1299.99 }
    ],
    total: 1299.99,
    status: "delivered",
    createdAt: new Date("2024-01-15")
  },
  {
    _id: 5002,
    userId: 2,
    items: [
      { productId: 1002, quantity: 2, price: 29.99 },
      { productId: 1003, quantity: 1, price: 12.99 }
    ],
    total: 72.97,
    status: "pending",
    createdAt: new Date()
  }
]);

// Créer des index
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ username: 1 }, { unique: true });
db.products.createIndex({ category: 1 });
db.products.createIndex({ tags: 1 });
db.orders.createIndex({ userId: 1 });
db.orders.createIndex({ createdAt: -1 });

print("Base de données initialisée avec succès !");
print("Utilisateurs:", db.users.countDocuments());
print("Produits:", db.products.countDocuments());
print("Commandes:", db.orders.countDocuments());
```

#### Exécution

```bash
# Exécuter le script
mongosh < init-data.js

# Ou depuis mongosh
mongosh
load("init-data.js")
```

### Script Makefile pour automatisation

```makefile
# Makefile pour gérer MongoDB en développement

.PHONY: help start stop restart logs shell clean seed

help: ## Afficher l'aide
	@echo "Commandes disponibles:"
	@echo "  make start   - Démarrer MongoDB"
	@echo "  make stop    - Arrêter MongoDB"
	@echo "  make restart - Redémarrer MongoDB"
	@echo "  make logs    - Afficher les logs"
	@echo "  make shell   - Ouvrir mongosh"
	@echo "  make seed    - Peupler avec des données de test"
	@echo "  make clean   - Supprimer toutes les données"

start: ## Démarrer MongoDB
	docker-compose up -d
	@echo "MongoDB démarré sur mongodb://localhost:27017"
	@echo "Mongo Express disponible sur http://localhost:8081"

stop: ## Arrêter MongoDB
	docker-compose down

restart: stop start ## Redémarrer MongoDB

logs: ## Afficher les logs
	docker-compose logs -f mongodb

shell: ## Ouvrir mongosh
	mongosh mongodb://localhost:27017/myapp

seed: ## Peupler la base de données
	mongosh < scripts/init-data.js
	@echo "Données de test insérées"

clean: ## Supprimer toutes les données
	docker-compose down -v
	@echo "Toutes les données supprimées"
```

---

## Outils de développement

### MongoDB Compass (GUI officiel)

#### Installation

```bash
# Ubuntu
wget https://downloads.mongodb.com/compass/mongodb-compass_latest_amd64.deb
sudo dpkg -i mongodb-compass_latest_amd64.deb

# macOS
brew install --cask mongodb-compass

# Windows : Télécharger depuis https://www.mongodb.com/products/compass
```

#### Connexion

```
mongodb://localhost:27017
```

#### Fonctionnalités utiles

- **Schema Analyzer** : Visualiser la structure des documents
- **Query Builder** : Construire des requêtes visuellement
- **Aggregation Pipeline Builder** : Créer des pipelines d'agrégation
- **Index Performance** : Analyser les performances des index
- **Explain Plans** : Comprendre l'exécution des requêtes

### mongosh (Shell moderne)

#### Configuration personnalisée

```javascript
// ~/.mongoshrc.js - Configuration du shell

// Prompt personnalisé
prompt = function() {
  return "[" + db.getName() + "] > ";
}

// Helpers personnalisés
function showCollections() {
  return db.getCollectionNames();
}

function clearCollection(collName) {
  return db[collName].deleteMany({});
}

function countAll() {
  const collections = db.getCollectionNames();
  collections.forEach(coll => {
    print(coll + ": " + db[coll].countDocuments());
  });
}

// Raccourcis
c = db.users;  // Alias pour db.users
```

#### Commandes rapides

```javascript
// Afficher toutes les bases
show dbs

// Utiliser une base
use myapp

// Collections
show collections

// Compter
db.users.countDocuments()

// Dernier document
db.users.findOne({}, {sort: {_id: -1}})

// Effacer une collection
db.users.deleteMany({})

// Statistiques
db.stats()
db.users.stats()
```

### Mongo Express (Web UI)

Déjà configuré dans le Docker Compose ci-dessus.

**URL** : http://localhost:8081

**Fonctionnalités** :
- Visualiser les bases de données et collections
- Exécuter des requêtes
- Insérer/modifier/supprimer des documents
- Exporter des données (JSON, CSV)

### VS Code Extensions

#### Extensions recommandées

```json
// .vscode/extensions.json
{
  "recommendations": [
    "mongodb.mongodb-vscode",      // MongoDB for VS Code (officiel)
    "dbaeumer.vscode-eslint",      // ESLint
    "esbenp.prettier-vscode"       // Prettier
  ]
}
```

#### Configuration MongoDB for VS Code

```json
// .vscode/settings.json
{
  "mongodb.connections": [
    {
      "name": "Local Dev",
      "connectionString": "mongodb://localhost:27017",
      "databases": ["myapp"]
    }
  ]
}
```

---

## Exemples de configuration par langage

### Node.js

#### package.json

```json
{
  "name": "myapp",
  "version": "1.0.0",
  "scripts": {
    "dev": "nodemon server.js",
    "seed": "node scripts/seed.js"
  },
  "dependencies": {
    "mongodb": "^6.3.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

#### Configuration de connexion

```javascript
// config/database.js
const { MongoClient } = require('mongodb');

const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017';
const dbName = process.env.DB_NAME || 'myapp';

let client;
let db;

async function connect() {
  if (db) return db;

  client = new MongoClient(uri);
  await client.connect();
  db = client.db(dbName);

  console.log(`Connecté à MongoDB: ${dbName}`);
  return db;
}

async function disconnect() {
  if (client) {
    await client.close();
    console.log('Déconnecté de MongoDB');
  }
}

module.exports = { connect, disconnect };
```

#### .env (développement)

```bash
# .env
MONGODB_URI=mongodb://localhost:27017
DB_NAME=myapp
NODE_ENV=development
```

### Python

#### requirements.txt

```txt
pymongo==4.6.1
python-dotenv==1.0.0
```

#### Configuration de connexion

```python
# config/database.py
import os
from pymongo import MongoClient
from dotenv import load_dotenv

load_dotenv()

MONGODB_URI = os.getenv('MONGODB_URI', 'mongodb://localhost:27017')
DB_NAME = os.getenv('DB_NAME', 'myapp')

client = MongoClient(MONGODB_URI)
db = client[DB_NAME]

def get_database():
    """Retourne la connexion à la base de données"""
    return db

def close_connection():
    """Ferme la connexion"""
    client.close()
    print("Déconnecté de MongoDB")
```

#### .env (développement)

```bash
# .env
MONGODB_URI=mongodb://localhost:27017
DB_NAME=myapp
ENVIRONMENT=development
```

### Java (Spring Boot)

#### application-dev.yml

```yaml
# src/main/resources/application-dev.yml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/myapp
      auto-index-creation: true

logging:
  level:
    org.springframework.data.mongodb: DEBUG

server:
  port: 8080
```

#### Configuration Bean

```java
// config/MongoConfig.java
package com.example.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.config.AbstractMongoClientConfiguration;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;

@Configuration
@EnableMongoRepositories(basePackages = "com.example.repository")
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Override
    protected String getDatabaseName() {
        return "myapp";
    }

    @Override
    protected boolean autoIndexCreation() {
        return true;  // Créer les index automatiquement en dev
    }
}
```

---

## Patterns de développement

### Hot Reload / Watch Mode

#### Nodemon (Node.js)

```json
// nodemon.json
{
  "watch": ["src"],
  "ext": "js,json",
  "ignore": ["src/**/*.test.js"],
  "exec": "node src/server.js"
}
```

#### Python (watchdog)

```python
# dev_server.py
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import subprocess
import time

class RestartHandler(FileSystemEventHandler):
    def on_modified(self, event):
        print(f"Détecté: {event.src_path}")
        # Redémarrer l'application

if __name__ == "__main__":
    observer = Observer()
    observer.schedule(RestartHandler(), "src", recursive=True)
    observer.start()
    # Démarrer l'app
```

### Tests unitaires avec base de données de test

#### Jest (Node.js)

```javascript
// tests/setup.js
const { MongoClient } = require('mongodb');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongod;
let connection;
let db;

beforeAll(async () => {
  // Démarrer serveur MongoDB en mémoire
  mongod = await MongoMemoryServer.create();
  const uri = mongod.getUri();

  connection = await MongoClient.connect(uri);
  db = connection.db('test');
});

afterAll(async () => {
  await connection.close();
  await mongod.stop();
});

beforeEach(async () => {
  // Nettoyer les collections avant chaque test
  const collections = await db.collections();
  for (let collection of collections) {
    await collection.deleteMany({});
  }
});

module.exports = { getDb: () => db };
```

#### Pytest (Python)

```python
# tests/conftest.py
import pytest
from pymongo import MongoClient
from mongomock import MongoClient as MockMongoClient

@pytest.fixture(scope="function")
def mongodb():
    """Base de données MongoDB mock pour les tests"""
    client = MockMongoClient()
    db = client.test_database
    yield db
    client.close()

@pytest.fixture(scope="function")
def sample_users(mongodb):
    """Insérer des utilisateurs de test"""
    mongodb.users.insert_many([
        {"_id": 1, "name": "John", "email": "john@test.com"},
        {"_id": 2, "name": "Jane", "email": "jane@test.com"}
    ])
    return mongodb.users
```

---

## Debugging

### Logs détaillés

```yaml
# mongod.conf pour debugging
systemLog:
  verbosity: 2  # 0-5, plus élevé = plus verbeux
  component:
    query:
      verbosity: 2
    write:
      verbosity: 2
```

### Profiler MongoDB

```javascript
// Activer le profiler en mode dev
use myapp

// Niveau 2 = toutes les opérations
db.setProfilingLevel(2)

// Voir les opérations lentes
db.system.profile.find().limit(10).sort({ts: -1}).pretty()

// Analyser une requête spécifique
db.users.find({email: "john@example.com"}).explain("executionStats")
```

### Monitoring léger

```javascript
// Script de monitoring simple
// monitor.js

setInterval(() => {
  const status = db.serverStatus();

  console.log("=== MongoDB Status ===");
  console.log("Connexions:", status.connections.current);
  console.log("Operations/sec:", status.opcounters.query + status.opcounters.insert);
  console.log("Memory (MB):", Math.round(status.mem.resident));
  console.log("======================");
}, 5000);
```

---

## Checklist de développement

### Configuration initiale

- [ ] MongoDB installé ou Docker configuré
- [ ] Connection string testée
- [ ] Base de données créée
- [ ] Données de test insérées
- [ ] Index de développement créés

### Outils

- [ ] mongosh installé et configuré
- [ ] MongoDB Compass installé
- [ ] Extension VS Code configurée
- [ ] Scripts d'initialisation prêts

### Code

- [ ] Driver MongoDB ajouté aux dépendances
- [ ] Fichier de configuration créé
- [ ] Variables d'environnement définies
- [ ] Connexion/déconnexion gérée proprement
- [ ] Gestion des erreurs en place

### Tests

- [ ] Base de données de test configurée
- [ ] Tests unitaires avec mock/memory MongoDB
- [ ] Fixtures de données créées
- [ ] Tests d'intégration fonctionnels

### Documentation

- [ ] README avec instructions de setup
- [ ] Scripts npm/make documentés
- [ ] Schéma des collections documenté
- [ ] Exemples de requêtes fournis

---

## Dépannage rapide

### MongoDB ne démarre pas

```bash
# Vérifier le processus
ps aux | grep mongod

# Vérifier les logs
tail -f /var/log/mongodb/mongod.log

# Vérifier le port
netstat -tuln | grep 27017

# Réinitialiser les données (ATTENTION: perte de données)
sudo rm -rf /var/lib/mongodb/*
sudo systemctl restart mongod
```

### Impossible de se connecter

```bash
# Tester la connexion
mongosh --eval "db.version()"

# Vérifier bind_ip
grep bindIp /etc/mongod.conf

# Doit être 127.0.0.1 ou 0.0.0.0

# Redémarrer après modification
sudo systemctl restart mongod
```

### Docker : Port déjà utilisé

```bash
# Trouver ce qui utilise le port 27017
sudo lsof -i :27017

# Arrêter le processus
sudo kill -9 <PID>

# Ou changer le port dans docker-compose.yml
ports:
  - "27018:27017"  # MongoDB sur port 27018
```

### Données corrompues

```bash
# Standalone local
mongod --dbpath /var/lib/mongodb --repair

# Docker
docker exec -it mongodb-dev mongod --repair
docker restart mongodb-dev
```

---

## Bonnes pratiques de développement

### ✅ À faire

- Utiliser des variables d'environnement
- Créer des scripts de seed reproductibles
- Versionner le schéma avec le code
- Utiliser des données de test réalistes
- Activer les logs verbeux
- Documenter les requêtes complexes

### ❌ À éviter

- Ne pas versionner les données de développement
- Ne pas hardcoder les credentials (même en dev)
- Ne pas utiliser de vraies données utilisateur
- Ne pas oublier d'indexer les champs requis
- Ne jamais mettre la config dev en production

---

## Passage en production

### Checklist avant déploiement

```javascript
// dev-to-prod-checklist.js

// ❌ Dev
const devConfig = {
  auth: false,              // ✅ Prod: true
  bindIp: "0.0.0.0",        // ✅ Prod: IP privée uniquement
  verbosity: 2,             // ✅ Prod: 0
  cacheSizeGB: 1,           // ✅ Prod: dimensionné
  replication: false,       // ✅ Prod: Replica Set
  backup: false,            // ✅ Prod: sauvegarde automatique
  monitoring: false,        // ✅ Prod: monitoring actif
  ssl: false,               // ✅ Prod: TLS activé
  firewall: "disabled"      // ✅ Prod: règles strictes
};
```

### Migration des données

```bash
# Exporter depuis dev
mongodump --uri="mongodb://localhost:27017/myapp" --out=./backup

# Importer en production (après configuration complète)
mongorestore --uri="mongodb://user:pass@prod-server:27017/myapp" ./backup/myapp
```

---

## Ressources

### Documentation

- [MongoDB Manual](https://docs.mongodb.com/manual/)
- [Docker Hub MongoDB](https://hub.docker.com/_/mongo)
- [MongoDB Drivers](https://docs.mongodb.com/drivers/)

### Outils

- [MongoDB Compass](https://www.mongodb.com/products/compass)
- [mongosh](https://www.mongodb.com/docs/mongodb-shell/)
- [MongoDB for VS Code](https://marketplace.visualstudio.com/items?itemName=mongodb.mongodb-vscode)
- [Mongo Express](https://github.com/mongo-express/mongo-express)

### Commandes rapides

```bash
# Alias utiles (.bashrc / .zshrc)
alias mdb="mongosh"
alias mdb-start="docker-compose up -d"
alias mdb-stop="docker-compose down"
alias mdb-logs="docker-compose logs -f mongodb"
alias mdb-shell="mongosh mongodb://localhost:27017/myapp"
```

---

**Configuration optimisée pour** : Développement local rapide
**Versions testées** : MongoDB 6.0+, 7.0+, 8.0+

⏭️ [Configuration haute performance](/annexes/configuration-reference/04-configuration-haute-performance.md)
