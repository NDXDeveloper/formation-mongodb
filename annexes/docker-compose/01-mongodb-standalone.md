🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.1 - MongoDB Standalone

## Introduction

Configuration Docker Compose pour déployer une **instance MongoDB unique** (standalone). Idéal pour le développement local, les tests unitaires et le prototypage rapide.

### 🎯 Cas d'usage

- Développement local d'applications
- Tests unitaires et intégration
- Prototypage et POC
- Formation et apprentissage MongoDB
- Environnements de démonstration

### ⚠️ Limitations

```markdown
❌ Pas de haute disponibilité
❌ Pas de redondance des données
❌ Pas de failover automatique
❌ Ne convient PAS à la production
✅ Parfait pour dev/test
```

---

## Configuration Minimale

### 🚀 Démarrage Rapide (1 minute)

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

**Commandes** :
```bash
# Démarrer
docker-compose up -d

# Vérifier
docker-compose ps

# Se connecter
mongosh mongodb://localhost:27017

# Arrêter
docker-compose down

# Arrêter et supprimer les données
docker-compose down -v
```

---

## Configuration Standard (Développement)

### 📦 Configuration Complète avec Authentification

**Structure de projet** :
```
mongodb-standalone/
├── docker-compose.yml
├── .env
├── .gitignore
├── init-scripts/
│   ├── 01-create-users.js
│   └── 02-seed-data.js
├── data/           # Créé automatiquement
└── logs/           # Créé automatiquement
```

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:${MONGO_VERSION:-7.0}
    container_name: mongodb-dev
    restart: unless-stopped

    ports:
      - "${MONGO_PORT:-27017}:27017"

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_DATABASE:-mydb}

    volumes:
      # Données persistantes
      - mongodb_data:/data/db

      # Scripts d'initialisation
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

      # Logs (optionnel)
      - ./logs:/var/log/mongodb

    networks:
      - mongodb_network

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  mongodb_data:
    driver: local

networks:
  mongodb_network:
    driver: bridge
```

**.env** :
```bash
# Version MongoDB
MONGO_VERSION=7.0

# Port d'exposition
MONGO_PORT=27017

# Authentification
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=SecurePassword123!
MONGO_DATABASE=mydb
```

**.gitignore** :
```
.env
data/
logs/
*.log
```

---

### 📜 Scripts d'Initialisation

**init-scripts/01-create-users.js** :
```javascript
// Exécuté automatiquement au premier démarrage
print('Creating users...');

// Switch to the application database
db = db.getSiblingDB('mydb');

// Create application user
db.createUser({
  user: 'appuser',
  pwd: 'apppassword',
  roles: [
    { role: 'readWrite', db: 'mydb' }
  ]
});

// Create read-only user
db.createUser({
  user: 'readonly',
  pwd: 'readonlypassword',
  roles: [
    { role: 'read', db: 'mydb' }
  ]
});

print('Users created successfully');
```

**init-scripts/02-seed-data.js** :
```javascript
print('Seeding initial data...');

db = db.getSiblingDB('mydb');

// Create collections
db.createCollection('users');
db.createCollection('products');
db.createCollection('orders');

// Insert sample data
db.users.insertMany([
  {
    name: 'John Doe',
    email: 'john@example.com',
    role: 'admin',
    createdAt: new Date()
  },
  {
    name: 'Jane Smith',
    email: 'jane@example.com',
    role: 'user',
    createdAt: new Date()
  }
]);

db.products.insertMany([
  {
    name: 'Product A',
    price: 29.99,
    stock: 100,
    category: 'electronics'
  },
  {
    name: 'Product B',
    price: 49.99,
    stock: 50,
    category: 'electronics'
  }
]);

// Create indexes
db.users.createIndex({ email: 1 }, { unique: true });
db.products.createIndex({ category: 1, price: 1 });

print('Data seeded successfully');
print('Users collection: ' + db.users.countDocuments());
print('Products collection: ' + db.products.countDocuments());
```

---

### 🚀 Utilisation

```bash
# 1. Créer le projet
mkdir mongodb-standalone && cd mongodb-standalone

# 2. Créer les fichiers
# (copier docker-compose.yml, .env, etc.)

# 3. Créer le dossier des scripts
mkdir -p init-scripts

# 4. Démarrer
docker-compose up -d

# 5. Voir les logs
docker-compose logs -f

# 6. Attendre que le healthcheck soit OK
docker-compose ps

# 7. Se connecter
mongosh "mongodb://admin:SecurePassword123!@localhost:27017/admin"

# 8. Tester l'user applicatif
mongosh "mongodb://appuser:apppassword@localhost:27017/mydb"

# 9. Vérifier les données
mongosh "mongodb://appuser:apppassword@localhost:27017/mydb" \
  --eval "db.users.find().pretty()"
```

---

## Configuration Avancée

### ⚙️ Avec Fichier de Configuration MongoDB

**mongod.conf** :
```yaml
# mongod.conf
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  verbosity: 0

storage:
  dbPath: /data/db
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 1000

security:
  authorization: enabled

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

setParameter:
  enableLocalhostAuthBypass: false
```

**docker-compose.yml** (avec config) :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb-dev
    restart: unless-stopped

    command: ["mongod", "--config", "/etc/mongod.conf"]

    ports:
      - "27017:27017"

    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: SecurePassword123!
      MONGO_INITDB_DATABASE: mydb

    volumes:
      - mongodb_data:/data/db
      - ./mongod.conf:/etc/mongod.conf:ro
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - ./logs:/var/log/mongodb

    networks:
      - mongodb_network

    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G

    healthcheck:
      test: |
        mongosh --username admin --password SecurePassword123! \
          --authenticationDatabase admin \
          --eval "db.adminCommand('ping')" || exit 1
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  mongodb_data:

networks:
  mongodb_network:
    driver: bridge
```

---

## Variantes de Configuration

### 🧪 Mode Test (Sans Persistance)

Idéal pour tests unitaires où les données sont temporaires :

```yaml
version: '3.8'

services:
  mongodb-test:
    image: mongo:7.0
    container_name: mongodb-test

    # Données en mémoire (tmpfs) - pas de persistance
    tmpfs:
      - /data/db

    environment:
      MONGO_INITDB_ROOT_USERNAME: test
      MONGO_INITDB_ROOT_PASSWORD: test

    ports:
      - "27017:27017"

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      timeout: 3s
      retries: 3
```

**Utilisation CI/CD** :
```yaml
# .gitlab-ci.yml ou .github/workflows
services:
  - mongo:7.0

variables:
  MONGO_INITDB_ROOT_USERNAME: test
  MONGO_INITDB_ROOT_PASSWORD: test

test:
  script:
    - npm test
```

---

### 🔓 Mode Développement (Sans Auth)

Pour développement très rapide (⚠️ jamais en prod) :

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb-noauth

    ports:
      - "27017:27017"

    volumes:
      - mongodb_data:/data/db

    # Pas d'authentification
    command: mongod --noauth

volumes:
  mongodb_data:
```

**Connexion** :
```bash
mongosh mongodb://localhost:27017
```

---

### 📊 Avec Monitoring (Mongo Express)

Interface web pour visualiser les données :

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: unless-stopped

    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password

    volumes:
      - mongodb_data:/data/db

    networks:
      - mongodb_network

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped

    ports:
      - "8081:8081"

    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_URL: mongodb://admin:password@mongodb:27017/
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: admin

    depends_on:
      - mongodb

    networks:
      - mongodb_network

volumes:
  mongodb_data:

networks:
  mongodb_network:
    driver: bridge
```

**Accès** :
- MongoDB : `mongodb://admin:password@localhost:27017`
- Mongo Express : `http://localhost:8081` (admin/admin)

---

### 🔐 Mode Production-Like

Configuration sécurisée pour environnements de staging :

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0.5  # Version fixe
    container_name: mongodb-staging
    restart: unless-stopped

    # Bind sur localhost uniquement
    ports:
      - "127.0.0.1:27017:27017"

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}

    volumes:
      - mongodb_data:/data/db
      - ./mongod.conf:/etc/mongod.conf:ro
      - ./logs:/var/log/mongodb

    command: ["mongod", "--config", "/etc/mongod.conf", "--auth"]

    networks:
      - mongodb_network

    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4G
        reservations:
          cpus: '2'
          memory: 2G

    healthcheck:
      test: |
        mongosh --username ${MONGO_ROOT_USERNAME} \
          --password ${MONGO_ROOT_PASSWORD} \
          --authenticationDatabase admin \
          --eval "db.adminCommand('ping')" || exit 1
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"

volumes:
  mongodb_data:
    driver: local

networks:
  mongodb_network:
    driver: bridge
```

---

## Connexion depuis Applications

### 🔗 Connection Strings

**Avec authentification** :
```bash
# Format générique
mongodb://username:password@host:port/database?authSource=admin

# Exemples
mongodb://admin:SecurePassword123!@localhost:27017/admin
mongodb://appuser:apppassword@localhost:27017/mydb

# Avec options
mongodb://appuser:apppassword@localhost:27017/mydb?retryWrites=true&w=majority
```

**Sans authentification** :
```bash
mongodb://localhost:27017
mongodb://localhost:27017/mydb
```

---

### 💻 Depuis le Code

**Node.js (MongoDB Driver)** :
```javascript
const { MongoClient } = require('mongodb');

const uri = process.env.MONGODB_URI ||
  'mongodb://appuser:apppassword@localhost:27017/mydb';

const client = new MongoClient(uri, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  maxPoolSize: 10
});

async function connect() {
  try {
    await client.connect();
    console.log('Connected to MongoDB');

    const db = client.db('mydb');
    const users = await db.collection('users').find().toArray();
    console.log(users);
  } catch (error) {
    console.error('Connection error:', error);
  } finally {
    await client.close();
  }
}

connect();
```

**Python (PyMongo)** :
```python
from pymongo import MongoClient
import os

uri = os.getenv('MONGODB_URI',
  'mongodb://appuser:apppassword@localhost:27017/mydb')

client = MongoClient(uri)

try:
    db = client.mydb
    users = list(db.users.find())
    print(users)
finally:
    client.close()
```

**Docker Compose avec Application** :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodb_data:/data/db
    networks:
      - app_network

  app:
    build: .
    environment:
      MONGODB_URI: mongodb://admin:password@mongodb:27017/mydb
    depends_on:
      - mongodb
    networks:
      - app_network

volumes:
  mongodb_data:

networks:
  app_network:
    driver: bridge
```

---

## Backup et Restauration

### 💾 Backup Manuel

**Avec mongodump** :
```bash
# Backup complet
docker exec mongodb mongodump \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  --out=/backup

# Copier le backup sur l'hôte
docker cp mongodb:/backup ./backup-$(date +%Y%m%d)

# Backup d'une base spécifique
docker exec mongodb mongodump \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  --db=mydb \
  --out=/backup
```

**Backup du volume Docker** :
```bash
# Méthode 1 : avec tar
docker run --rm \
  -v mongodb_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mongodb-backup-$(date +%Y%m%d).tar.gz /data

# Méthode 2 : avec rsync
docker run --rm \
  -v mongodb_data:/data:ro \
  -v $(pwd)/backup:/backup \
  instrumentisto/rsync-ssh \
  rsync -avz /data/ /backup/
```

---

### 🔄 Restauration

**Avec mongorestore** :
```bash
# Copier le backup dans le conteneur
docker cp ./backup-20240101 mongodb:/backup

# Restaurer
docker exec mongodb mongorestore \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  /backup

# Restaurer une base spécifique
docker exec mongodb mongorestore \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  --db=mydb \
  /backup/mydb
```

**Restauration du volume** :
```bash
# Arrêter le conteneur
docker-compose down

# Restaurer le volume
docker run --rm \
  -v mongodb_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/mongodb-backup-20240101.tar.gz --strip 1"

# Redémarrer
docker-compose up -d
```

---

### 🤖 Backup Automatisé

**Script de backup** :
```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="./backups"
DATE=$(date +%Y%m%d-%H%M%S)
CONTAINER_NAME="mongodb"

mkdir -p $BACKUP_DIR

echo "Starting backup at $DATE..."

# Backup avec mongodump
docker exec $CONTAINER_NAME mongodump \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  --gzip \
  --archive=/backup/mongodb-$DATE.gz

# Copier sur l'hôte
docker cp $CONTAINER_NAME:/backup/mongodb-$DATE.gz $BACKUP_DIR/

# Nettoyer dans le conteneur
docker exec $CONTAINER_NAME rm /backup/mongodb-$DATE.gz

# Supprimer les backups > 7 jours
find $BACKUP_DIR -name "mongodb-*.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/mongodb-$DATE.gz"
```

**Cron job** :
```bash
# Éditer crontab
crontab -e

# Backup quotidien à 2h du matin
0 2 * * * /path/to/backup.sh >> /var/log/mongodb-backup.log 2>&1
```

---

## Maintenance

### 🧹 Nettoyage

```bash
# Voir l'espace utilisé
docker system df
docker volume ls

# Arrêter et supprimer le conteneur
docker-compose down

# Supprimer les volumes (⚠️ PERTE DE DONNÉES)
docker-compose down -v

# Nettoyer les images inutilisées
docker image prune -a

# Nettoyer tout (conteneurs arrêtés, volumes, réseaux, images)
docker system prune -a --volumes
```

---

### 📊 Monitoring

**Logs** :
```bash
# Suivre les logs en temps réel
docker-compose logs -f mongodb

# Dernières 100 lignes
docker-compose logs --tail=100 mongodb

# Logs depuis un temps donné
docker-compose logs --since 30m mongodb

# Filtrer les logs
docker-compose logs mongodb | grep ERROR
```

**Statistiques** :
```bash
# Stats en temps réel
docker stats mongodb

# Inspection du conteneur
docker inspect mongodb

# Processus dans le conteneur
docker top mongodb

# Espace disque du volume
docker system df -v | grep mongodb_data
```

**Healthcheck** :
```bash
# Status du healthcheck
docker inspect --format='{{.State.Health.Status}}' mongodb

# Détails du healthcheck
docker inspect mongodb | jq '.[0].State.Health'
```

---

### 🔄 Mise à Jour MongoDB

```bash
# 1. Backup complet (IMPORTANT)
./backup.sh

# 2. Arrêter le conteneur
docker-compose down

# 3. Modifier la version dans docker-compose.yml
# image: mongo:7.0  →  image: mongo:8.0

# 4. Pull de la nouvelle image
docker-compose pull

# 5. Redémarrer
docker-compose up -d

# 6. Vérifier les logs
docker-compose logs -f

# 7. Vérifier la version
docker exec mongodb mongosh --eval "db.version()"

# 8. Si problème, rollback
docker-compose down
# Revenir à l'ancienne version dans docker-compose.yml
docker-compose up -d
# Restaurer le backup si nécessaire
```

---

## Dépannage

### 🔧 Problèmes Courants

#### Conteneur ne démarre pas

```bash
# Voir les logs détaillés
docker-compose logs mongodb

# Vérifier si le port est libre
sudo lsof -i :27017
sudo netstat -tuln | grep 27017

# Tuer le processus qui utilise le port
sudo kill -9 $(sudo lsof -t -i:27017)

# Vérifier les permissions du volume
ls -la ./data
sudo chown -R 999:999 ./data  # UID MongoDB
```

#### Erreur d'authentification

```bash
# Recréer avec volumes propres
docker-compose down -v
docker-compose up -d

# Vérifier les variables d'environnement
docker exec mongodb env | grep MONGO

# Se connecter en tant que root
mongosh "mongodb://admin:password@localhost:27017/admin"

# Lister les users
db.getSiblingDB('admin').system.users.find()
```

#### Performance lente

```bash
# Vérifier les ressources
docker stats mongodb

# Augmenter le cache WiredTiger
# Dans docker-compose.yml :
command: mongod --wiredTigerCacheSizeGB 2

# Vérifier les index
docker exec -it mongodb mongosh -u admin -p password --authenticationDatabase admin
> use mydb
> db.collection.getIndexes()
> db.collection.find().explain("executionStats")
```

#### Volume corrompu

```bash
# Arrêter le conteneur
docker-compose down

# Sauvegarder le volume existant
docker run --rm \
  -v mongodb_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/corrupted-volume.tar.gz /data

# Supprimer et recréer
docker volume rm mongodb_data
docker-compose up -d

# Restaurer depuis un backup propre
# (voir section Backup et Restauration)
```

---

## Checklist de Déploiement

### ✅ Avant le Démarrage

```markdown
□ Docker et Docker Compose installés et à jour
□ Port 27017 libre (ou modifier MONGO_PORT)
□ .env créé avec mots de passe forts
□ .env ajouté à .gitignore
□ Scripts d'init préparés (si nécessaire)
□ Backup strategy définie
□ Ressources suffisantes (2 GB RAM minimum)
□ docker-compose.yml vérifié (version, config)
```

### ✅ Après le Démarrage

```markdown
□ Conteneur démarré : docker-compose ps
□ Healthcheck PASSING : docker ps
□ Connexion fonctionne : mongosh
□ Authentification OK
□ Scripts d'init exécutés : vérifier les collections
□ Volumes persistants : docker volume ls
□ Backup testé : backup + restore
□ Documentation à jour : README.md
```

---

## Exemples Complets

### 📁 Projet Node.js avec MongoDB

**Structure** :
```
my-app/
├── docker-compose.yml
├── .env
├── .gitignore
├── package.json
├── src/
│   └── index.js
└── init-scripts/
    └── 01-init.js
```

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
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
      retries: 3

  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      MONGODB_URI: mongodb://${MONGO_ROOT_USERNAME}:${MONGO_ROOT_PASSWORD}@mongodb:27017/${MONGO_DATABASE}?authSource=admin
    depends_on:
      mongodb:
        condition: service_healthy
    volumes:
      - ./src:/app/src
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
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=SecurePass123!
MONGO_DATABASE=myapp
```

**src/index.js** :
```javascript
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
const port = 3000;

const client = new MongoClient(process.env.MONGODB_URI);

app.get('/', async (req, res) => {
  try {
    await client.connect();
    const db = client.db();
    const count = await db.collection('users').countDocuments();
    res.json({ users: count });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(port, () => {
  console.log(`App running on port ${port}`);
});
```

**Utilisation** :
```bash
docker-compose up -d
curl http://localhost:3000
```

---

## Ressources

### 📚 Documentation
- [MongoDB Docker Hub](https://hub.docker.com/_/mongo)
- [MongoDB Manual](https://www.mongodb.com/docs/manual/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

### 🔗 Liens Utiles
- [MongoDB Connection String](https://www.mongodb.com/docs/manual/reference/connection-string/)
- [mongodump Documentation](https://www.mongodb.com/docs/database-tools/mongodump/)
- [mongorestore Documentation](https://www.mongodb.com/docs/database-tools/mongorestore/)

---

## Conclusion

Cette configuration MongoDB Standalone est **idéale pour** :
- ✅ Développement local rapide
- ✅ Tests unitaires et d'intégration
- ✅ Prototypage et POC
- ✅ Formation et apprentissage

**Ne pas utiliser pour** :
- ❌ Production (utiliser Replica Set minimum)
- ❌ Applications critiques nécessitant HA
- ❌ Données sensibles sans backup automatisé

**Prochaine étape** : F.2 - Replica Set avec Docker Compose

---

**Version** : 1.0
**Testé avec** : MongoDB 7.0, Docker 24.0, Docker Compose 2.23

⏭️ [Replica Set avec Docker Compose](/annexes/docker-compose/02-replica-set-docker-compose.md)
