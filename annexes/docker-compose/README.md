🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F. Docker et Docker Compose

## Vue d'ensemble

Cette annexe fournit des **configurations Docker et Docker Compose prêtes à l'emploi** pour déployer MongoDB dans différentes architectures. Ces configurations sont optimisées pour le développement, les tests et la production.

---

## Objectif et Utilisation

### 🎯 Objectif

Fournir des configurations Docker Compose **testées et documentées** pour :
- Démarrer rapidement un environnement MongoDB local
- Tester des architectures distribuées (Replica Set, Sharding)
- Prototyper des applications
- Environnements de développement isolés et reproductibles

### 📋 Public Cible

- **Développeurs** : Environnement local MongoDB
- **DevOps** : Infrastructure as Code pour MongoDB
- **QA/Test** : Environnements de test reproductibles
- **Formation** : Apprentissage des architectures MongoDB

---

## Prérequis

### 🔧 Installation Docker

```bash
# Vérifier l'installation
docker --version
docker-compose --version

# Minimum requis
Docker Engine: 20.10+
Docker Compose: 2.0+
```

**Installation rapide** :
```bash
# Linux (Ubuntu/Debian)
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Ajouter user au groupe docker
sudo usermod -aG docker $USER
newgrp docker

# Docker Compose (si non inclus)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Vérifier
docker run hello-world
```

### 💾 Ressources Système

| Configuration | RAM | CPU | Disque |
|--------------|-----|-----|--------|
| **Standalone** | 2 GB | 2 cores | 10 GB |
| **Replica Set (3 membres)** | 6 GB | 4 cores | 30 GB |
| **Sharded Cluster** | 8-12 GB | 8 cores | 50 GB |

---

## Structure de l'Annexe

Cette annexe contient **4 configurations complètes** :

### F.1 - MongoDB Standalone
**Objectif** : Instance MongoDB unique pour développement

**Cas d'usage** :
- Développement local
- Tests unitaires
- Prototypage rapide
- Apprentissage MongoDB

**Composants** :
- 1 conteneur MongoDB
- Volumes persistants
- Configuration basique

---

### F.2 - Replica Set avec Docker Compose
**Objectif** : Cluster de réplication à 3 membres

**Cas d'usage** :
- Tests de haute disponibilité
- Simulation de failover
- Développement d'applications HA
- Formation réplication

**Composants** :
- 3 conteneurs MongoDB (Primary + 2 Secondaries)
- Réseau privé Docker
- Script d'initialisation automatique
- Volumes persistants par membre

---

### F.3 - Sharded Cluster avec Docker Compose
**Objectif** : Architecture shardée complète

**Cas d'usage** :
- Tests de scalabilité horizontale
- Développement pour grandes données
- Formation sharding
- POC architectures distribuées

**Composants** :
- 3+ Shards (chacun un Replica Set)
- 3 Config Servers (CSRS)
- 2+ Mongos (query routers)
- Scripts d'initialisation
- Réseau dédié

---

### F.4 - Stack Complète (MongoDB + Outils)
**Objectif** : Environnement complet avec interfaces graphiques

**Cas d'usage** :
- Environnement de développement complet
- Démos et présentations
- Formation avec interfaces visuelles
- Prototypage d'applications

**Composants** :
- MongoDB (Standalone ou RS)
- Mongo Express (Web UI)
- Application exemple (Node.js/Python)
- Reverse proxy (optionnel)
- Monitoring (optionnel)

---

## Concepts de Base

### 🐳 Docker pour MongoDB

**Image officielle** :
```bash
# Versions disponibles
docker pull mongo:latest          # Dernière version
docker pull mongo:8.0             # Version 8.0
docker pull mongo:7.0             # Version 7.0
docker pull mongo:6.0             # Version 6.0

# Images spécifiques
docker pull mongo:8.0-jammy       # Ubuntu Jammy
docker pull mongo:8.0-ubi8        # Red Hat UBI
```

**Variables d'environnement clés** :
```yaml
environment:
  MONGO_INITDB_ROOT_USERNAME: admin
  MONGO_INITDB_ROOT_PASSWORD: password
  MONGO_INITDB_DATABASE: mydb
```

---

### 📦 Docker Compose

**Structure de base** :
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    networks:
      - mongodb_network

volumes:
  mongodb_data:

networks:
  mongodb_network:
    driver: bridge
```

---

## Commandes Essentielles

### 🚀 Gestion des Conteneurs

```bash
# Démarrer tous les services
docker-compose up -d

# Démarrer avec rebuild
docker-compose up -d --build

# Voir les logs
docker-compose logs -f
docker-compose logs -f mongodb  # Service spécifique

# Arrêter les services
docker-compose stop

# Arrêter et supprimer
docker-compose down

# Arrêter et supprimer avec volumes
docker-compose down -v

# Redémarrer un service
docker-compose restart mongodb

# Status des services
docker-compose ps
```

### 🔍 Débogage

```bash
# Se connecter au conteneur
docker exec -it mongodb bash
docker exec -it mongodb mongosh

# Voir les logs en temps réel
docker logs -f mongodb

# Inspecter le conteneur
docker inspect mongodb

# Statistiques en temps réel
docker stats mongodb

# Voir les processus
docker top mongodb

# Copier fichiers depuis/vers conteneur
docker cp mongodb:/data/backup.gz ./
docker cp ./config.js mongodb:/tmp/
```

### 🗄️ Gestion des Volumes

```bash
# Lister les volumes
docker volume ls

# Inspecter un volume
docker volume inspect mongodb_data

# Supprimer un volume
docker volume rm mongodb_data

# Supprimer volumes non utilisés
docker volume prune

# Backup d'un volume
docker run --rm -v mongodb_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/mongodb-backup.tar.gz /data
```

---

## Réseaux Docker

### 🌐 Configuration Réseau

**Types de réseaux** :
```yaml
networks:
  # Bridge (défaut) - isolé
  mongodb_network:
    driver: bridge

  # Host - utilise réseau de l'hôte
  mongodb_host:
    driver: host

  # Overlay - multi-host (Swarm)
  mongodb_overlay:
    driver: overlay
```

**Résolution DNS** :
```yaml
services:
  mongodb1:
    hostname: mongodb1
    networks:
      - mongodb_network

  mongodb2:
    hostname: mongodb2
    networks:
      - mongodb_network

# mongodb1 et mongodb2 se résolvent mutuellement par nom
```

**Ports et exposition** :
```yaml
ports:
  - "27017:27017"        # host:container
  - "127.0.0.1:27017:27017"  # bind sur localhost uniquement

expose:
  - "27017"              # Exposé aux autres conteneurs, pas à l'hôte
```

---

## Volumes et Persistance

### 💾 Types de Volumes

**Named volumes (recommandé)** :
```yaml
volumes:
  mongodb_data:  # Géré par Docker

services:
  mongodb:
    volumes:
      - mongodb_data:/data/db
```

**Bind mounts** :
```yaml
services:
  mongodb:
    volumes:
      - ./data:/data/db              # Données
      - ./config/mongod.conf:/etc/mongod.conf:ro  # Config (read-only)
      - ./init:/docker-entrypoint-initdb.d  # Scripts init
```

**Tmpfs (temporaire en RAM)** :
```yaml
services:
  mongodb:
    tmpfs:
      - /tmp
```

### 📂 Points de Montage MongoDB

```yaml
volumes:
  # Données
  - mongodb_data:/data/db

  # Configuration
  - ./mongod.conf:/etc/mongod.conf:ro

  # Logs
  - ./logs:/var/log/mongodb

  # Scripts d'initialisation
  - ./init-scripts:/docker-entrypoint-initdb.d:ro

  # Backup
  - ./backups:/backup
```

---

## Configuration MongoDB dans Docker

### ⚙️ Fichier de Configuration

**mongod.conf** :
```yaml
# mongod.conf
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

storage:
  dbPath: /data/db
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1

net:
  port: 27017
  bindIp: 0.0.0.0

security:
  authorization: enabled

replication:
  replSetName: rs0
```

**Utilisation** :
```yaml
services:
  mongodb:
    image: mongo:7.0
    command: ["mongod", "--config", "/etc/mongod.conf"]
    volumes:
      - ./mongod.conf:/etc/mongod.conf:ro
```

---

### 🔐 Authentification et Sécurité

**Avec authentification** :
```yaml
services:
  mongodb:
    image: mongo:7.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    command: mongod --auth
```

**Variables d'environnement (.env)** :
```bash
# .env
MONGO_VERSION=7.0
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=SuperSecretPassword123!
MONGO_DATABASE=mydb
```

```yaml
# docker-compose.yml
services:
  mongodb:
    image: mongo:${MONGO_VERSION}
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
```

**⚠️ Sécurité** :
```bash
# .env dans .gitignore
echo ".env" >> .gitignore

# Générer mot de passe sécurisé
openssl rand -base64 32
```

---

## Scripts d'Initialisation

### 📜 Scripts Automatiques

**Structure** :
```
project/
├── docker-compose.yml
└── init-scripts/
    ├── 01-create-users.js
    ├── 02-create-collections.js
    └── 03-seed-data.js
```

**Exemple de script** :
```javascript
// init-scripts/01-create-users.js
db = db.getSiblingDB('mydb');

db.createUser({
  user: 'appuser',
  pwd: 'apppassword',
  roles: [
    { role: 'readWrite', db: 'mydb' }
  ]
});

db.createCollection('users');
db.createCollection('orders');

print('Database initialized successfully');
```

**Montage** :
```yaml
services:
  mongodb:
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
```

**⚠️ Important** :
```markdown
- Scripts exécutés SEULEMENT si /data/db est vide
- Ordre alphabétique d'exécution
- Extensions supportées : .js, .sh
- Pas de réexécution sur restart
```

---

## Health Checks

### 🏥 Vérification de Santé

```yaml
services:
  mongodb:
    image: mongo:7.0
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 40s
```

**Avec authentification** :
```yaml
healthcheck:
  test: |
    mongosh --username admin --password ${MONGO_PASSWORD} --authenticationDatabase admin --eval "db.adminCommand('ping')" || exit 1
  interval: 10s
  timeout: 5s
  retries: 3
```

**Utilisation** :
```bash
# Voir le status health
docker ps

# Format spécifique
docker inspect --format='{{.State.Health.Status}}' mongodb
```

---

## Bonnes Pratiques

### ✅ Configuration Générale

```markdown
1. **Versions explicites**
   ✅ image: mongo:7.0.5
   ❌ image: mongo:latest

2. **Restart policy**
   ✅ restart: unless-stopped
   ⚠️ restart: always (peut masquer des problèmes)

3. **Ressources limitées**
   deploy:
     resources:
       limits:
         cpus: '2'
         memory: 4G

4. **Healthchecks systématiques**
   healthcheck: [...]

5. **Logs rotatifs**
   logging:
     driver: "json-file"
     options:
       max-size: "10m"
       max-file: "3"

6. **Named volumes pour données**
   ✅ volumes: mongodb_data:/data/db
   ❌ volumes: ./data:/data/db (sauf dev)
```

---

### 🔒 Sécurité

```markdown
1. **Jamais de mots de passe en clair**
   ✅ Utiliser .env
   ✅ Docker secrets (Swarm)
   ❌ Hardcoded dans docker-compose.yml

2. **Réseau dédié**
   ✅ networks: mongodb_network (driver: bridge)
   ❌ network_mode: host (expose tout)

3. **Ports minimaux**
   ✅ expose: "27017" (inter-conteneurs)
   ⚠️ ports: "27017:27017" (seulement si nécessaire)
   ✅ ports: "127.0.0.1:27017:27017" (localhost uniquement)

4. **Read-only où possible**
   volumes:
     - ./mongod.conf:/etc/mongod.conf:ro

5. **User non-root (si possible)**
   user: "999:999"  # mongodb user/group

6. **.gitignore complet**
   .env
   data/
   logs/
   backups/
```

---

### 📊 Performance

```markdown
1. **Volumes performants**
   ✅ Named volumes (overlay2)
   ⚠️ Bind mounts (plus lent)

2. **Ressources CPU/RAM suffisantes**
   deploy.resources.limits

3. **Cache WiredTiger approprié**
   command: --wiredTigerCacheSizeGB 2

4. **Pas de swap dans conteneurs**
   deploy:
     resources:
       reservations:
         memory: 4G

5. **SSD pour stockage**
   Sur l'hôte Docker
```

---

### 🧹 Maintenance

```markdown
1. **Cleanup régulier**
   docker system prune -a
   docker volume prune

2. **Backup volumes**
   Scripts automatisés

3. **Updates contrôlés**
   Test en staging avant prod

4. **Monitoring**
   docker stats
   Prometheus/Grafana

5. **Documentation**
   README.md avec instructions
```

---

## Patterns d'Utilisation

### 🔄 Développement Local

```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  mongodb:
    image: mongo:7.0
    ports:
      - "27017:27017"  # Exposé pour outils locaux
    environment:
      MONGO_INITDB_ROOT_USERNAME: dev
      MONGO_INITDB_ROOT_PASSWORD: dev123
    volumes:
      - mongodb_dev:/data/db
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

volumes:
  mongodb_dev:
```

```bash
# Utilisation
docker-compose -f docker-compose.dev.yml up -d
```

---

### 🧪 Tests / CI/CD

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  mongodb:
    image: mongo:7.0
    tmpfs:
      - /data/db  # En mémoire, pas de persistance
    environment:
      MONGO_INITDB_ROOT_USERNAME: test
      MONGO_INITDB_ROOT_PASSWORD: test
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      timeout: 3s
      retries: 3
```

---

### 🏭 Production-like

```yaml
# docker-compose.prod.yml
version: '3.8'
services:
  mongodb:
    image: mongo:7.0.5  # Version fixe
    restart: unless-stopped
    ports:
      - "127.0.0.1:27017:27017"  # Localhost uniquement
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    volumes:
      - mongodb_prod:/data/db
      - ./mongod.conf:/etc/mongod.conf:ro
      - ./logs:/var/log/mongodb
    command: ["mongod", "--config", "/etc/mongod.conf", "--auth"]
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
    healthcheck:
      test: |
        mongosh --username ${MONGO_ROOT_USERNAME} --password ${MONGO_ROOT_PASSWORD} \
          --authenticationDatabase admin --eval "db.adminCommand('ping')" || exit 1
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"
    networks:
      - mongodb_network

volumes:
  mongodb_prod:
    driver: local

networks:
  mongodb_network:
    driver: bridge
```

---

## Dépannage

### 🔧 Problèmes Courants

#### Conteneur ne démarre pas

```bash
# Voir les logs
docker-compose logs mongodb

# Causes fréquentes :
# 1. Port 27017 déjà utilisé
sudo lsof -i :27017
sudo netstat -tuln | grep 27017

# 2. Volume corrompu
docker-compose down -v  # ⚠️ Supprime les données
docker-compose up -d

# 3. Permissions
sudo chown -R 999:999 ./data

# 4. Mémoire insuffisante
docker stats
```

#### Connexion refuse

```bash
# Vérifier le conteneur
docker ps
docker inspect mongodb | grep IPAddress

# Tester la connexion
docker exec -it mongodb mongosh
docker exec -it mongodb mongosh -u admin -p password --authenticationDatabase admin

# Depuis l'hôte
mongosh mongodb://admin:password@localhost:27017/admin
```

#### Performance lente

```bash
# Statistiques
docker stats mongodb

# Vérifier I/O
docker exec -it mongodb iostat -x 1

# Logs de performance
docker exec -it mongodb mongosh --eval "db.setProfilingLevel(1, {slowms: 100})"
```

---

## Migrations et Backup

### 💾 Backup de Volumes

```bash
# Backup
docker run --rm \
  -v mongodb_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mongodb-$(date +%Y%m%d).tar.gz /data

# Restore
docker run --rm \
  -v mongodb_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/mongodb-20240101.tar.gz --strip 1"
```

### 🔄 Migration entre Environnements

```bash
# Export depuis conteneur
docker exec mongodb mongodump --out=/backup --username=admin --password=pass
docker cp mongodb:/backup ./backup-$(date +%Y%m%d)

# Import vers nouveau conteneur
docker cp ./backup-20240101 mongodb-new:/backup
docker exec mongodb-new mongorestore /backup --username=admin --password=pass
```

---

## Checklist de Déploiement

### ✅ Avant le Déploiement

```markdown
□ Version MongoDB fixe (pas :latest)
□ Variables d'environnement dans .env
□ .env dans .gitignore
□ Mot de passe fort et unique
□ Healthcheck configuré
□ Ressources limitées (deploy.resources)
□ Restart policy appropriée
□ Logs rotatifs configurés
□ Réseau dédié
□ Volumes nommés pour données
□ Scripts d'init testés (si applicable)
□ Backup strategy définie
□ Documentation README.md
```

### ✅ Après le Déploiement

```markdown
□ Healthcheck PASSING
□ Connexion fonctionnelle
□ Authentification testée
□ Scripts d'init exécutés correctement
□ Volumes persistants vérifiés
□ Performance acceptable
□ Logs accessibles
□ Monitoring en place
□ Test de backup/restore
□ Documentation à jour
```

---

## Outils et Intégrations

### 🔧 Outils Visuels

**Mongo Express** :
```yaml
services:
  mongo-express:
    image: mongo-express:latest
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_URL: mongodb://admin:password@mongodb:27017/
    depends_on:
      - mongodb
```

**MongoDB Compass** :
```bash
# Connection string pour Compass
mongodb://admin:password@localhost:27017/?authSource=admin
```

---

### 📊 Monitoring

**Prometheus + Grafana** :
```yaml
services:
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    ports:
      - "9216:9216"
    environment:
      MONGODB_URI: mongodb://admin:password@mongodb:27017
    depends_on:
      - mongodb
```

---

## Structure de cette Annexe

Cette annexe contient les 4 configurations suivantes :

1. **[F.1 - MongoDB Standalone](./01-mongodb-standalone.md)**
   - Configuration basique
   - Développement local
   - Tests unitaires

2. **[F.2 - Replica Set](./02-replica-set-docker-compose.md)**
   - 3 membres
   - Haute disponibilité
   - Tests de failover

3. **[F.3 - Sharded Cluster](./03-sharded-cluster-docker-compose.md)**
   - Architecture complète
   - Scalabilité horizontale
   - Configuration avancée

4. **[F.4 - Stack Complète](./04-stack-complete.md)**
   - MongoDB + Mongo Express
   - Application exemple
   - Environnement complet

---

## Ressources Complémentaires

### Documentation Officielle
- [Docker Hub - MongoDB](https://hub.docker.com/_/mongo)
- [MongoDB Docker Documentation](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-community-with-docker/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

### Exemples Communautaires
- [MongoDB Docker GitHub](https://github.com/docker-library/mongo)
- [Awesome MongoDB Docker](https://github.com/topics/mongodb-docker)

### Sécurité
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [MongoDB Security Checklist](https://www.mongodb.com/docs/manual/administration/security-checklist/)

---

## Notes Importantes

### ⚠️ Limitations Docker

```markdown
1. **Performance I/O** : Légèrement inférieure au natif
2. **Réseau** : Latence minimale entre conteneurs
3. **Production** : Privilégier Kubernetes pour orchestration avancée
4. **Backup** : Stratégie adaptée aux volumes Docker
5. **Updates** : Planifier les migrations de version
```

### 💡 Recommandations

```markdown
✅ Dev/Test : Docker Compose parfait
✅ Staging : Docker Compose acceptable
⚠️ Production : Considérer Kubernetes ou cloud managé (Atlas)
✅ CI/CD : Excellent pour tests automatisés
✅ Formation : Idéal pour apprendre MongoDB
```

---

**Version** : 1.0
**Compatible avec** : MongoDB 6.x, 7.x, 8.x | Docker 20.10+ | Docker Compose 2.0+

⏭️ [MongoDB standalone](/annexes/docker-compose/01-mongodb-standalone.md)
