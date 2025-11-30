🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.9 Installation via Docker

## Introduction

Docker est une plateforme de conteneurisation qui permet d'exécuter des applications dans des environnements isolés appelés **conteneurs**. Installer MongoDB via Docker présente de nombreux avantages, notamment la simplicité, la portabilité et l'isolation.

Cette méthode est particulièrement recommandée pour :
- Le développement local
- Les tests et le prototypage
- Les environnements d'intégration continue (CI/CD)
- L'apprentissage de MongoDB

---

## Pourquoi utiliser Docker pour MongoDB ?

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Avantages de Docker pour MongoDB                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ✅ Installation en une commande                                   │
│   ✅ Pas de configuration système complexe                          │
│   ✅ Isolation complète (n'affecte pas votre système)               │
│   ✅ Facilité de mise à jour et changement de version               │
│   ✅ Environnement reproductible                                    │
│   ✅ Nettoyage facile (supprimer le conteneur suffit)               │
│   ✅ Possibilité d'exécuter plusieurs versions en parallèle         │
│   ✅ Configuration identique entre développeurs d'une équipe        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Comparaison : Installation native vs Docker

| Aspect | Installation native | Docker |
|--------|---------------------|--------|
| **Temps d'installation** | 10-30 minutes | 2-5 minutes |
| **Configuration système** | Requise | Aucune |
| **Isolation** | Non | Complète |
| **Changement de version** | Complexe | Simple |
| **Nettoyage** | Manuel | `docker rm` |
| **Portabilité** | Limitée | Totale |
| **Multi-versions** | Difficile | Facile |

---

## Prérequis : Installer Docker

Avant de commencer, vous devez avoir Docker installé sur votre machine.

### Installation de Docker

#### Windows

1. Téléchargez **Docker Desktop** depuis [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
2. Exécutez l'installateur
3. Redémarrez votre ordinateur si demandé
4. Lancez Docker Desktop

> **Note** : Docker Desktop pour Windows nécessite WSL 2 (Windows Subsystem for Linux) ou Hyper-V.

#### macOS

1. Téléchargez **Docker Desktop** depuis [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
2. Ouvrez le fichier `.dmg` et glissez Docker dans Applications
3. Lancez Docker depuis les Applications

Ou via Homebrew :

```bash
brew install --cask docker
```

#### Linux (Ubuntu/Debian)

```bash
# Mettre à jour les paquets
sudo apt-get update

# Installer les prérequis
sudo apt-get install -y ca-certificates curl gnupg

# Ajouter la clé GPG officielle de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajouter le dépôt Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installer Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Ajouter votre utilisateur au groupe docker (pour éviter sudo)
sudo usermod -aG docker $USER

# Appliquer les changements (ou déconnectez-vous et reconnectez-vous)
newgrp docker
```

### Vérifier l'installation de Docker

```bash
# Vérifier la version de Docker
docker --version

# Vérifier que Docker fonctionne
docker run hello-world
```

Résultat attendu :

```
Docker version 24.x.x, build xxxxxxx

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## Lancer MongoDB avec Docker

### Commande de base

La façon la plus simple de lancer MongoDB avec Docker :

```bash
docker run --name mongodb -d -p 27017:27017 mongo:latest
```

Décomposons cette commande :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Anatomie de la commande                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   docker run                                                        │
│   │                                                                 │
│   ├── --name mongodb      → Nom du conteneur                        │
│   │                                                                 │
│   ├── -d                  → Mode détaché (arrière-plan)             │
│   │                                                                 │
│   ├── -p 27017:27017      → Mapping de port                         │
│   │   │      │              (hôte:conteneur)                        │
│   │   │      └── Port dans le conteneur                             │
│   │   └── Port sur votre machine                                    │
│   │                                                                 │
│   └── mongo:latest        → Image Docker à utiliser                 │
│       │     │                                                       │
│       │     └── Tag (version)                                       │
│       └── Nom de l'image                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Vérifier que le conteneur tourne

```bash
# Lister les conteneurs en cours d'exécution
docker ps

# Résultat attendu
CONTAINER ID   IMAGE          COMMAND                  STATUS          PORTS                      NAMES
a1b2c3d4e5f6   mongo:latest   "docker-entrypoint.s…"   Up 2 minutes    0.0.0.0:27017->27017/tcp   mongodb
```

### Se connecter à MongoDB

```bash
# Depuis votre machine (si mongosh est installé)
mongosh

# Ou via Docker (sans avoir mongosh installé localement)
docker exec -it mongodb mongosh
```

---

## Choisir une version spécifique

### Tags disponibles

L'image officielle MongoDB propose plusieurs tags (versions) :

| Tag | Description |
|-----|-------------|
| `mongo:latest` | Dernière version stable |
| `mongo:8.0` | Version 8.0.x (dernière 8.0) |
| `mongo:7.0` | Version 7.0.x |
| `mongo:6.0` | Version 6.0.x |
| `mongo:5.0` | Version 5.0.x |
| `mongo:8.0.4` | Version exacte 8.0.4 |

### Exemples

```bash
# Dernière version
docker run --name mongodb -d -p 27017:27017 mongo:latest

# Version 7.0 spécifiquement
docker run --name mongodb7 -d -p 27017:27017 mongo:7.0

# Version exacte
docker run --name mongodb-specific -d -p 27017:27017 mongo:8.0.4
```

### Exécuter plusieurs versions en parallèle

```bash
# MongoDB 8.0 sur le port 27017
docker run --name mongo8 -d -p 27017:27017 mongo:8.0

# MongoDB 7.0 sur le port 27018
docker run --name mongo7 -d -p 27018:27017 mongo:7.0

# MongoDB 6.0 sur le port 27019
docker run --name mongo6 -d -p 27019:27017 mongo:6.0
```

```
┌─────────────────────────────────────────────────────────────────────┐
│              Plusieurs versions MongoDB en parallèle                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Votre machine                                                     │
│   │                                                                 │
│   ├── Port 27017 ──────► Conteneur mongo8 (MongoDB 8.0)             │
│   │                                                                 │
│   ├── Port 27018 ──────► Conteneur mongo7 (MongoDB 7.0)             │
│   │                                                                 │
│   └── Port 27019 ──────► Conteneur mongo6 (MongoDB 6.0)             │
│                                                                     │
│   Connexion :                                                       │
│   mongosh --port 27017    → MongoDB 8.0                             │
│   mongosh --port 27018    → MongoDB 7.0                             │
│   mongosh --port 27019    → MongoDB 6.0                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Persistance des données avec les volumes

### Le problème : données éphémères

Par défaut, les données stockées dans un conteneur Docker sont **perdues** lorsque le conteneur est supprimé.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Sans volume (données perdues)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   1. Créer le conteneur                                             │
│      docker run --name mongodb -d mongo                             │
│                                                                     │
│   2. Insérer des données                                            │
│      db.users.insertOne({ name: "Alice" })                          │
│                                                                     │
│   3. Supprimer le conteneur                                         │
│      docker rm -f mongodb                                           │
│                                                                     │
│   4. Recréer le conteneur                                           │
│      docker run --name mongodb -d mongo                             │
│                                                                     │
│   5. Les données ont DISPARU ! 😱                                   │
│      db.users.find()  →  Aucun résultat                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### La solution : les volumes Docker

Les **volumes** permettent de stocker les données en dehors du conteneur, sur votre système hôte.

```bash
# Créer un volume nommé
docker volume create mongodb_data

# Lancer MongoDB avec le volume
docker run --name mongodb \
  -d \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  mongo:latest
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Avec volume (données persistantes)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Votre machine                        Conteneur MongoDB            │
│   ┌─────────────────────┐             ┌─────────────────────┐       │
│   │                     │             │                     │       │
│   │   Volume Docker     │◄───────────►│    /data/db         │       │
│   │   "mongodb_data"    │   Montage   │  (données MongoDB)  │       │
│   │                     │             │                     │       │
│   │   📁 Données        │             │                     │       │
│   │   persistantes      │             │                     │       │
│   │                     │             │                     │       │
│   └─────────────────────┘             └─────────────────────┘       │
│                                                                     │
│   Même si le conteneur est supprimé, les données restent            │
│   dans le volume et seront disponibles au prochain lancement.       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Utiliser un répertoire local (bind mount)

Vous pouvez aussi monter un répertoire spécifique de votre machine :

```bash
# Créer un répertoire local pour les données
mkdir -p ~/mongodb-data

# Lancer MongoDB avec le bind mount
docker run --name mongodb \
  -d \
  -p 27017:27017 \
  -v ~/mongodb-data:/data/db \
  mongo:latest
```

> **Note** : Sur Windows, utilisez un chemin comme `C:/Users/VotreNom/mongodb-data:/data/db`

### Commandes de gestion des volumes

```bash
# Lister les volumes
docker volume ls

# Inspecter un volume
docker volume inspect mongodb_data

# Supprimer un volume (attention : supprime les données !)
docker volume rm mongodb_data

# Supprimer les volumes non utilisés
docker volume prune
```

---

## Configuration avancée avec variables d'environnement

### Activer l'authentification

Pour sécuriser MongoDB avec un utilisateur administrateur :

```bash
docker run --name mongodb \
  -d \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=motdepasse123 \
  mongo:latest
```

### Variables d'environnement disponibles

| Variable | Description |
|----------|-------------|
| `MONGO_INITDB_ROOT_USERNAME` | Nom d'utilisateur admin |
| `MONGO_INITDB_ROOT_PASSWORD` | Mot de passe admin |
| `MONGO_INITDB_DATABASE` | Base de données à créer au démarrage |

### Se connecter avec authentification

```bash
# Via Docker
docker exec -it mongodb mongosh -u admin -p motdepasse123

# Depuis votre machine
mongosh "mongodb://admin:motdepasse123@localhost:27017"
```

### Script d'initialisation personnalisé

Vous pouvez exécuter des scripts au premier démarrage du conteneur :

```bash
# Créer un répertoire pour les scripts
mkdir -p ~/mongo-init

# Créer un script d'initialisation
cat > ~/mongo-init/init.js << 'EOF'
// Créer une base de données et un utilisateur
db = db.getSiblingDB('mabase');

db.createUser({
  user: 'appuser',
  pwd: 'apppassword',
  roles: [
    { role: 'readWrite', db: 'mabase' }
  ]
});

// Insérer des données initiales
db.config.insertOne({
  app: 'MonApplication',
  version: '1.0.0',
  initialized: new Date()
});

print('Base de données initialisée avec succès !');
EOF

# Lancer MongoDB avec le script
docker run --name mongodb \
  -d \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  -v ~/mongo-init:/docker-entrypoint-initdb.d:ro \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=motdepasse123 \
  mongo:latest
```

Les scripts dans `/docker-entrypoint-initdb.d/` sont exécutés automatiquement au premier démarrage (fichiers `.js` ou `.sh`).

---

## Docker Compose

### Qu'est-ce que Docker Compose ?

**Docker Compose** est un outil qui permet de définir et gérer des applications multi-conteneurs à l'aide d'un fichier YAML. C'est idéal pour :

- Définir la configuration de manière déclarative
- Partager la configuration avec une équipe
- Reproduire l'environnement facilement

### Installation de Docker Compose

Docker Compose est inclus avec Docker Desktop (Windows/macOS). Sur Linux, il est généralement installé avec le plugin `docker-compose-plugin`.

```bash
# Vérifier l'installation
docker compose version
```

### Fichier docker-compose.yml de base

Créez un fichier nommé `docker-compose.yml` :

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:8.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: motdepasse123

volumes:
  mongodb_data:
```

### Lancer avec Docker Compose

```bash
# Démarrer les services (en arrière-plan)
docker compose up -d

# Voir les logs
docker compose logs -f

# Arrêter les services
docker compose down

# Arrêter et supprimer les volumes (attention aux données !)
docker compose down -v
```

### Configuration complète avec Mongo Express

**Mongo Express** est une interface web pour administrer MongoDB. Voici une configuration complète :

```yaml
version: '3.8'

services:
  # Service MongoDB
  mongodb:
    image: mongo:8.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d:ro
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: motdepasse123
    networks:
      - mongodb_network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Interface web Mongo Express
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: motdepasse123
      ME_CONFIG_MONGODB_URL: mongodb://admin:motdepasse123@mongodb:27017/
      ME_CONFIG_BASICAUTH_USERNAME: webadmin
      ME_CONFIG_BASICAUTH_PASSWORD: webpassword
    networks:
      - mongodb_network
    depends_on:
      mongodb:
        condition: service_healthy

networks:
  mongodb_network:
    driver: bridge

volumes:
  mongodb_data:
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Architecture Docker Compose                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Votre navigateur                                                  │
│         │                                                           │
│         │ http://localhost:8081                                     │
│         ▼                                                           │
│   ┌─────────────────┐                                               │
│   │  Mongo Express  │ ◄─── Interface web d'administration           │
│   │   (port 8081)   │                                               │
│   └────────┬────────┘                                               │
│            │                                                        │
│            │ Réseau Docker interne                                  │
│            │ (mongodb_network)                                      │
│            ▼                                                        │
│   ┌─────────────────┐                                               │
│   │    MongoDB      │ ◄─── Base de données                          │
│   │   (port 27017)  │                                               │
│   └────────┬────────┘                                               │
│            │                                                        │
│            ▼                                                        │
│   ┌─────────────────┐                                               │
│   │  Volume Docker  │ ◄─── Données persistantes                     │
│   │  mongodb_data   │                                               │
│   └─────────────────┘                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Accéder à Mongo Express

Après le lancement avec `docker compose up -d` :

1. Ouvrez votre navigateur
2. Allez sur `http://localhost:8081`
3. Connectez-vous avec :
   - Username : `webadmin`
   - Password : `webpassword`

---

## Commandes Docker essentielles

### Gestion des conteneurs

```bash
# Lister les conteneurs en cours d'exécution
docker ps

# Lister tous les conteneurs (y compris arrêtés)
docker ps -a

# Démarrer un conteneur arrêté
docker start mongodb

# Arrêter un conteneur
docker stop mongodb

# Redémarrer un conteneur
docker restart mongodb

# Supprimer un conteneur (doit être arrêté)
docker rm mongodb

# Forcer la suppression d'un conteneur en cours d'exécution
docker rm -f mongodb
```

### Logs et débogage

```bash
# Voir les logs
docker logs mongodb

# Suivre les logs en temps réel
docker logs -f mongodb

# Voir les dernières 100 lignes
docker logs --tail 100 mongodb

# Exécuter une commande dans le conteneur
docker exec -it mongodb bash

# Ouvrir le shell MongoDB
docker exec -it mongodb mongosh

# Avec authentification
docker exec -it mongodb mongosh -u admin -p motdepasse123
```

### Gestion des images

```bash
# Lister les images téléchargées
docker images

# Télécharger une image sans lancer de conteneur
docker pull mongo:8.0

# Supprimer une image
docker rmi mongo:8.0

# Supprimer les images non utilisées
docker image prune
```

### Informations et statistiques

```bash
# Informations détaillées sur un conteneur
docker inspect mongodb

# Statistiques en temps réel (CPU, mémoire, etc.)
docker stats mongodb

# Statistiques de tous les conteneurs
docker stats
```

---

## Configurations courantes

### Configuration pour le développement

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  mongodb:
    image: mongo:8.0
    container_name: mongodb-dev
    ports:
      - "27017:27017"
    volumes:
      - mongodb_dev_data:/data/db
    # Pas d'authentification pour simplifier le développement

volumes:
  mongodb_dev_data:
```

```bash
# Lancer en mode développement
docker compose -f docker-compose.dev.yml up -d
```

### Configuration avec limites de ressources

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:8.0
    container_name: mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

volumes:
  mongodb_data:
```

### Configuration avec fichier de config MongoDB

```bash
# Créer un fichier de configuration
mkdir -p ~/mongo-config
cat > ~/mongo-config/mongod.conf << 'EOF'
storage:
  dbPath: /data/db
  journal:
    enabled: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  port: 27017
  bindIp: 0.0.0.0

# Activer le profilage des requêtes lentes
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
EOF
```

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:8.0
    container_name: mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
      - ./mongo-config/mongod.conf:/etc/mongod.conf:ro
      - mongodb_logs:/var/log/mongodb
    command: ["mongod", "--config", "/etc/mongod.conf"]

volumes:
  mongodb_data:
  mongodb_logs:
```

---

## Dépannage

### Problème 1 : Le conteneur ne démarre pas

```bash
# Vérifier les logs
docker logs mongodb

# Causes courantes :
# - Port déjà utilisé
# - Problème de permissions sur le volume
# - Image corrompue
```

**Solutions :**

```bash
# Vérifier si le port est utilisé
lsof -i :27017          # Linux/macOS
netstat -ano | findstr :27017   # Windows

# Recréer le conteneur
docker rm -f mongodb
docker run --name mongodb -d -p 27017:27017 mongo:latest

# Retélécharger l'image
docker pull mongo:latest
```

### Problème 2 : Erreur de permission sur le volume

```bash
# Symptôme
Error: couldn't open /data/db/WiredTiger
```

**Solutions :**

```bash
# Linux : corriger les permissions
sudo chown -R 999:999 ~/mongodb-data

# Ou utiliser un volume Docker nommé (recommandé)
docker volume create mongodb_data
docker run -v mongodb_data:/data/db ...
```

### Problème 3 : Impossible de se connecter

```bash
# Vérifier que le conteneur tourne
docker ps

# Vérifier les ports exposés
docker port mongodb

# Tester la connexion depuis le conteneur
docker exec -it mongodb mongosh --eval "db.adminCommand('ping')"
```

### Problème 4 : Données perdues après redémarrage

**Cause** : Vous n'avez pas utilisé de volume.

**Solution** : Toujours utiliser un volume pour persister les données :

```bash
docker run -v mongodb_data:/data/db ...
```

### Problème 5 : Conteneur qui redémarre en boucle

```bash
# Voir les logs pour comprendre l'erreur
docker logs mongodb

# Vérifier l'état du conteneur
docker inspect mongodb | grep -A 5 "State"
```

---

## Bonnes pratiques

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Bonnes pratiques Docker + MongoDB                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   1. TOUJOURS utiliser des volumes                                  │
│      → Évite la perte de données                                    │
│                                                                     │
│   2. Spécifier une version explicite                                │
│      → mongo:8.0 plutôt que mongo:latest                            │
│      → Garantit la reproductibilité                                 │
│                                                                     │
│   3. Utiliser Docker Compose                                        │
│      → Configuration versionnable et partageable                    │
│                                                                     │
│   4. Activer l'authentification                                     │
│      → Même en développement (bonne habitude)                       │
│                                                                     │
│   5. Limiter les ressources                                         │
│      → Évite que MongoDB consomme toute la RAM                      │
│                                                                     │
│   6. Utiliser des health checks                                     │
│      → Détecte les problèmes automatiquement                        │
│                                                                     │
│   7. Nommer vos conteneurs et volumes                               │
│      → Facilite la gestion et le débogage                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Récapitulatif des commandes

### Commandes rapides

```bash
# Démarrage rapide (développement)
docker run --name mongodb -d -p 27017:27017 -v mongodb_data:/data/db mongo:8.0

# Avec authentification
docker run --name mongodb -d -p 27017:27017 \
  -v mongodb_data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo:8.0

# Se connecter au shell
docker exec -it mongodb mongosh

# Voir les logs
docker logs -f mongodb

# Arrêter
docker stop mongodb

# Redémarrer
docker start mongodb

# Supprimer
docker rm -f mongodb
```

### Docker Compose

```bash
# Démarrer
docker compose up -d

# Voir le statut
docker compose ps

# Voir les logs
docker compose logs -f

# Arrêter
docker compose down

# Arrêter et supprimer les volumes
docker compose down -v

# Reconstruire
docker compose up -d --build
```

---

## Conclusion

Docker simplifie considérablement l'installation et la gestion de MongoDB, particulièrement pour le développement et les tests. Les points essentiels à retenir :

- Une seule commande suffit pour démarrer MongoDB
- Les **volumes** sont essentiels pour persister les données
- **Docker Compose** facilite la gestion de configurations complexes
- Vous pouvez exécuter plusieurs versions de MongoDB en parallèle

Dans la prochaine section, nous découvrirons les outils graphiques et en ligne de commande pour interagir avec MongoDB : mongosh, MongoDB Compass et Atlas.

---

## Points clés à retenir

- `docker run -d -p 27017:27017 mongo` lance MongoDB en une commande
- Utilisez **toujours un volume** (`-v`) pour persister les données
- Les variables d'environnement configurent l'authentification
- **Docker Compose** est recommandé pour les projets
- `docker exec -it mongodb mongosh` ouvre le shell MongoDB
- Spécifiez une **version explicite** (ex: `mongo:8.0`)
- **Mongo Express** fournit une interface web pratique

---


⏭️ [Présentation des outils : mongosh, MongoDB Compass, Atlas](/01-introduction-a-mongodb/10-presentation-outils.md)
