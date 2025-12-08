🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.2 Déploiement avec Docker

## Introduction

Docker offre une approche moderne et portable pour déployer MongoDB, particulièrement adaptée aux environnements de développement, de test et aux déploiements cloud-native. Cette section couvre les pratiques avancées de conteneurisation MongoDB, de l'image standalone au cluster shardé complet.

La conteneurisation de MongoDB présente des défis spécifiques liés à la nature stateful de la base de données, nécessitant une attention particulière à la persistance des données, la performance I/O, et la gestion des réseaux.

---

## Architecture Docker pour MongoDB

### Modèle de Déploiement

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Deployment Model                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Development              Staging              Production       │
│  ┌──────────────┐        ┌──────────────┐    ┌──────────────┐   │
│  │   Standalone │        │ Replica Set  │    │   Sharded    │   │
│  │              │        │              │    │   Cluster    │   │
│  │  ┌────────┐  │        │  ┌────────┐  │    │              │   │
│  │  │MongoDB │  │        │  │Primary │  │    │  Multiple    │   │
│  │  │Container  │        │  └────────┘  │    │  Replica     │   │
│  │  └────────┘  │        │  ┌────────┐  │    │  Sets        │   │
│  │              │        │  │Secondary  │    │              │   │
│  │  Volume      │        │  └────────┘  │    │  + Config    │   │
│  │  Ephemeral   │        │  ┌────────┐  │    │  + Mongos    │   │
│  │  or Mounted  │        │  │Secondary  │    │              │   │
│  │              │        │  └────────┘  │    │  Volumes     │   │
│  └──────────────┘        │              │    │  Network     │   │
│                          │  Volumes     │    │  Monitoring  │   │
│                          │  Network     │    │              │   │
│                          └──────────────┘    └──────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Principes de Conteneurisation MongoDB

**1. Persistance des données**
- Les volumes Docker doivent être utilisés pour `/data/db`
- Snapshots et backups doivent être gérés au niveau du volume
- Les performances I/O dépendent du driver de stockage

**2. Networking**
- Les replica sets nécessitent une résolution DNS stable
- Les connexions inter-conteneurs doivent être optimisées
- Le service discovery doit être configuré correctement

**3. Configuration**
- Les configurations doivent être injectées via variables d'environnement ou ConfigMaps
- Les secrets doivent être gérés séparément (Docker Secrets, Vault)
- La configuration doit être immutable (rebuild plutôt que modifier)

---

## Images Docker MongoDB

### Images Officielles

MongoDB fournit des images officielles sur Docker Hub :

```bash
# Latest version
docker pull mongo:latest

# Specific version
docker pull mongo:7.0.5

# Enterprise version
docker pull mongodb/mongodb-enterprise-server:7.0.5-ubi8
```

**Structure de l'image officielle :**

```
mongo:7.0.5
├── Base: Ubuntu 22.04
├── MongoDB binaries
│   ├── mongod
│   ├── mongos
│   ├── mongosh
│   └── tools
├── Entrypoint: docker-entrypoint.sh
├── Default user: mongodb (999:999)
├── Default port: 27017
├── Data directory: /data/db
└── Config directory: /data/configdb
```

### Image Personnalisée

Création d'une image MongoDB optimisée pour la production :

```dockerfile
# Dockerfile.mongodb
FROM mongo:7.0.5

# Metadata
LABEL maintainer="platform-team@company.com"
LABEL version="7.0.5"
LABEL description="MongoDB production-ready image"

# Install additional tools
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    procps \
    net-tools \
    dnsutils \
    jq \
    python3-pip \
    && pip3 install --no-cache-dir \
    pymongo \
    requests \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install MongoDB monitoring tools
RUN curl -LO https://github.com/percona/mongodb_exporter/releases/download/v0.40.0/mongodb_exporter-0.40.0.linux-amd64.tar.gz \
    && tar xzf mongodb_exporter-0.40.0.linux-amd64.tar.gz \
    && mv mongodb_exporter-0.40.0.linux-amd64/mongodb_exporter /usr/local/bin/ \
    && rm -rf mongodb_exporter-*

# Custom scripts
COPY scripts/healthcheck.sh /usr/local/bin/healthcheck.sh
COPY scripts/backup.sh /usr/local/bin/backup.sh
COPY scripts/init-replica-set.sh /usr/local/bin/init-replica-set.sh

RUN chmod +x /usr/local/bin/*.sh

# Custom mongod configuration
COPY config/mongod.conf /etc/mongod.conf

# TLS certificates (will be overridden by volume in production)
RUN mkdir -p /etc/mongodb/ssl
COPY certs/ca.pem /etc/mongodb/ssl/ca.pem
COPY certs/mongodb.pem /etc/mongodb/ssl/mongodb.pem
RUN chown -R mongodb:mongodb /etc/mongodb/ssl \
    && chmod 400 /etc/mongodb/ssl/mongodb.pem

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD /usr/local/bin/healthcheck.sh

# Create backup directory
RUN mkdir -p /backup && chown mongodb:mongodb /backup

# Expose ports
EXPOSE 27017 9216

# Use custom entrypoint
COPY scripts/docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

USER mongodb

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["mongod", "--config", "/etc/mongod.conf"]
```

```bash
#!/bin/bash
# scripts/healthcheck.sh

set -eo pipefail

# MongoDB connection parameters
MONGO_HOST="${MONGO_HOST:-localhost}"
MONGO_PORT="${MONGO_PORT:-27017}"
MONGO_USER="${MONGO_INITDB_ROOT_USERNAME:-}"
MONGO_PASSWORD="${MONGO_INITDB_ROOT_PASSWORD:-}"

# Build connection string
if [ -n "$MONGO_USER" ] && [ -n "$MONGO_PASSWORD" ]; then
    AUTH="--username $MONGO_USER --password $MONGO_PASSWORD --authenticationDatabase admin"
else
    AUTH=""
fi

# Check if MongoDB is responsive
if mongosh --host "$MONGO_HOST" --port "$MONGO_PORT" $AUTH --eval "db.adminCommand('ping')" --quiet > /dev/null 2>&1; then
    # For replica set members, check replica set status
    if [ "$MONGO_REPLICA_SET" != "" ]; then
        RS_STATUS=$(mongosh --host "$MONGO_HOST" --port "$MONGO_PORT" $AUTH \
            --eval "rs.status().ok" --quiet 2>/dev/null || echo "0")

        if [ "$RS_STATUS" = "1" ]; then
            exit 0
        else
            echo "Replica set not initialized or unhealthy"
            exit 1
        fi
    else
        exit 0
    fi
else
    echo "MongoDB is not responsive"
    exit 1
fi
```

```bash
#!/bin/bash
# scripts/docker-entrypoint.sh

set -Eeuo pipefail

# Source original entrypoint functions
source /usr/local/bin/docker-entrypoint.sh

# Custom initialization logic
_custom_init() {
    echo "Running custom initialization..."

    # Set proper file permissions
    chown -R mongodb:mongodb /data/db /data/configdb

    # Initialize replica set if requested
    if [ "${MONGO_INITDB_REPLICA_SET:-}" ] && [ "${MONGO_INITDB_PRIMARY:-false}" = "true" ]; then
        echo "Waiting for MongoDB to start before initializing replica set..."
        sleep 10
        /usr/local/bin/init-replica-set.sh &
    fi

    # Start monitoring exporter if enabled
    if [ "${MONGODB_EXPORTER_ENABLED:-false}" = "true" ]; then
        echo "Starting MongoDB Exporter..."
        mongodb_exporter \
            --mongodb.uri="mongodb://localhost:27017" \
            --web.listen-address=":9216" \
            --collect-all &
    fi
}

# Run custom init before starting MongoDB
_custom_init

# Execute original entrypoint
exec "$@"
```

```bash
#!/bin/bash
# scripts/init-replica-set.sh

set -eo pipefail

REPLICA_SET="${MONGO_REPLICA_SET}"
MONGO_USER="${MONGO_INITDB_ROOT_USERNAME}"
MONGO_PASSWORD="${MONGO_INITDB_ROOT_PASSWORD}"

# Wait for MongoDB to be ready
until mongosh --eval "db.adminCommand('ping')" --quiet > /dev/null 2>&1; do
    echo "Waiting for MongoDB to start..."
    sleep 2
done

echo "Initializing replica set: $REPLICA_SET"

# Get replica set member hostnames from environment
# Expected format: REPLICA_SET_MEMBERS="mongo-0:27017,mongo-1:27017,mongo-2:27017"
IFS=',' read -ra MEMBERS <<< "${REPLICA_SET_MEMBERS}"

# Build replica set configuration
RS_CONFIG="{"
RS_CONFIG+="  _id: '$REPLICA_SET',"
RS_CONFIG+="  members: ["

for i in "${!MEMBERS[@]}"; do
    RS_CONFIG+="    { _id: $i, host: '${MEMBERS[$i]}' }"
    [ $i -lt $((${#MEMBERS[@]} - 1)) ] && RS_CONFIG+=","
done

RS_CONFIG+="  ]"
RS_CONFIG+="}"

# Initialize replica set
mongosh --eval "
    try {
        rs.initiate($RS_CONFIG);
        print('Replica set initialized successfully');
    } catch (e) {
        if (e.code === 23) {
            print('Replica set already initialized');
        } else {
            throw e;
        }
    }
"

# Wait for replica set to elect a primary
echo "Waiting for primary election..."
until mongosh --eval "rs.status().members.filter(m => m.stateStr === 'PRIMARY').length > 0" --quiet | grep -q "true"; do
    echo "Waiting for primary..."
    sleep 2
done

echo "Replica set initialized and primary elected"

# Create admin user if credentials provided
if [ -n "$MONGO_USER" ] && [ -n "$MONGO_PASSWORD" ]; then
    echo "Creating admin user..."
    mongosh admin --eval "
        db.createUser({
            user: '$MONGO_USER',
            pwd: '$MONGO_PASSWORD',
            roles: [
                { role: 'root', db: 'admin' }
            ]
        })
    " || echo "Admin user already exists"
fi
```

```yaml
# config/mongod.conf
# MongoDB Configuration for Docker

storage:
  dbPath: /data/db
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1  # Will be overridden by environment variable
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /data/db/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: 0
  component:
    replication:
      verbosity: 1

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

processManagement:
  timeZoneInfo: /usr/share/zoneinfo

# Security will be configured via environment variables and secrets
security:
  authorization: enabled

# Replication will be configured if MONGO_REPLICA_SET is set
# replication:
#   replSetName: rs0

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

setParameter:
  diagnosticDataCollectionEnabled: true
  enableLocalhostAuthBypass: false
```

### Build et Push

```bash
#!/bin/bash
# build.sh

set -euo pipefail

VERSION="7.0.5"
REGISTRY="docker.io/company"
IMAGE_NAME="mongodb"

# Build image
docker build \
    --build-arg MONGO_VERSION=$VERSION \
    --tag $REGISTRY/$IMAGE_NAME:$VERSION \
    --tag $REGISTRY/$IMAGE_NAME:latest \
    -f Dockerfile.mongodb .

# Security scanning
echo "Scanning image for vulnerabilities..."
docker scan $REGISTRY/$IMAGE_NAME:$VERSION || true

# Run tests
echo "Running container tests..."
./test-image.sh $REGISTRY/$IMAGE_NAME:$VERSION

# Push to registry
if [ "${PUSH_TO_REGISTRY:-false}" = "true" ]; then
    echo "Pushing to registry..."
    docker push $REGISTRY/$IMAGE_NAME:$VERSION
    docker push $REGISTRY/$IMAGE_NAME:latest
fi

echo "Build completed successfully!"
```

---

## Docker Compose : Topologies MongoDB

### Standalone MongoDB

```yaml
# docker-compose-standalone.yml
version: '3.9'

services:
  mongodb:
    image: mongo:7.0.5
    container_name: mongodb-standalone
    restart: unless-stopped

    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: app_db

    ports:
      - "27017:27017"

    volumes:
      # Data persistence
      - mongodb_data:/data/db
      - mongodb_config:/data/configdb

      # Custom configuration
      - ./config/mongod.conf:/etc/mongod.conf:ro

      # Initialization scripts
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

      # Backup directory
      - ./backup:/backup

    command: ["mongod", "--config", "/etc/mongod.conf"]

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

    networks:
      - mongodb_network

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G

    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"

  # MongoDB monitoring with Prometheus exporter
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb-exporter
    restart: unless-stopped

    environment:
      MONGODB_URI: "mongodb://admin:${MONGO_ROOT_PASSWORD}@mongodb:27017"

    ports:
      - "9216:9216"

    command:
      - "--collect-all"
      - "--compatible-mode"

    networks:
      - mongodb_network

    depends_on:
      mongodb:
        condition: service_healthy

volumes:
  mongodb_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/mongodb

  mongodb_config:
    driver: local

networks:
  mongodb_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Replica Set avec Docker Compose

```yaml
# docker-compose-replica-set.yml
version: '3.9'

x-mongo-common: &mongo-common
  image: mongo:7.0.5
  restart: unless-stopped

  environment: &mongo-env
    MONGO_INITDB_ROOT_USERNAME: admin
    MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    MONGO_REPLICA_SET: rs0

  command: >
    mongod
    --replSet rs0
    --bind_ip_all
    --keyFile /etc/mongodb/keyfile
    --auth

  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 60s

  networks:
    - mongodb_network

  logging:
    driver: "json-file"
    options:
      max-size: "50m"
      max-file: "3"

services:
  # Primary MongoDB node
  mongo-primary:
    <<: *mongo-common
    container_name: mongo-primary
    hostname: mongo-primary

    ports:
      - "27017:27017"

    volumes:
      - mongo_primary_data:/data/db
      - mongo_primary_config:/data/configdb
      - ./keyfile:/etc/mongodb/keyfile:ro
      - ./init-replica-set.sh:/docker-entrypoint-initdb.d/init-replica-set.sh:ro

    environment:
      <<: *mongo-env
      MONGO_INITDB_PRIMARY: "true"
      REPLICA_SET_MEMBERS: "mongo-primary:27017,mongo-secondary-1:27017,mongo-secondary-2:27017"

    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
      placement:
        constraints:
          - node.labels.mongodb.role == primary

  # Secondary MongoDB node 1
  mongo-secondary-1:
    <<: *mongo-common
    container_name: mongo-secondary-1
    hostname: mongo-secondary-1

    ports:
      - "27018:27017"

    volumes:
      - mongo_secondary_1_data:/data/db
      - mongo_secondary_1_config:/data/configdb
      - ./keyfile:/etc/mongodb/keyfile:ro

    environment:
      <<: *mongo-env

    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
      placement:
        constraints:
          - node.labels.mongodb.role == secondary

    depends_on:
      mongo-primary:
        condition: service_healthy

  # Secondary MongoDB node 2
  mongo-secondary-2:
    <<: *mongo-common
    container_name: mongo-secondary-2
    hostname: mongo-secondary-2

    ports:
      - "27019:27017"

    volumes:
      - mongo_secondary_2_data:/data/db
      - mongo_secondary_2_config:/data/configdb
      - ./keyfile:/etc/mongodb/keyfile:ro

    environment:
      <<: *mongo-env

    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
      placement:
        constraints:
          - node.labels.mongodb.role == secondary

    depends_on:
      mongo-primary:
        condition: service_healthy

  # Replica set initializer (runs once)
  mongo-init:
    image: mongo:7.0.5
    container_name: mongo-init

    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}

    volumes:
      - ./scripts/init-replica-set-compose.sh:/init-replica-set.sh:ro

    networks:
      - mongodb_network

    command: >
      bash -c "
        sleep 30 &&
        /init-replica-set.sh
      "

    depends_on:
      mongo-primary:
        condition: service_healthy
      mongo-secondary-1:
        condition: service_healthy
      mongo-secondary-2:
        condition: service_healthy

    restart: "no"

  # MongoDB monitoring
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb-exporter
    restart: unless-stopped

    environment:
      MONGODB_URI: "mongodb://admin:${MONGO_ROOT_PASSWORD}@mongo-primary:27017,mongo-secondary-1:27017,mongo-secondary-2:27017/?replicaSet=rs0"

    ports:
      - "9216:9216"

    command:
      - "--collect-all"
      - "--compatible-mode"
      - "--discovering-mode"

    networks:
      - mongodb_network

    depends_on:
      - mongo-primary
      - mongo-secondary-1
      - mongo-secondary-2

volumes:
  mongo_primary_data:
    driver: local
  mongo_primary_config:
    driver: local
  mongo_secondary_1_data:
    driver: local
  mongo_secondary_1_config:
    driver: local
  mongo_secondary_2_data:
    driver: local
  mongo_secondary_2_config:
    driver: local

networks:
  mongodb_network:
    driver: overlay
    attachable: true
    ipam:
      config:
        - subnet: 172.21.0.0/16
```

```bash
#!/bin/bash
# scripts/init-replica-set-compose.sh

set -euo pipefail

MONGO_USER="${MONGO_INITDB_ROOT_USERNAME}"
MONGO_PASSWORD="${MONGO_INITDB_ROOT_PASSWORD}"

echo "Waiting for all MongoDB instances to be ready..."
sleep 10

echo "Initializing replica set..."

mongosh "mongodb://$MONGO_USER:$MONGO_PASSWORD@mongo-primary:27017/admin" --eval "
  try {
    rs.initiate({
      _id: 'rs0',
      members: [
        { _id: 0, host: 'mongo-primary:27017', priority: 3 },
        { _id: 1, host: 'mongo-secondary-1:27017', priority: 2 },
        { _id: 2, host: 'mongo-secondary-2:27017', priority: 1 }
      ]
    });
    print('Replica set initiated successfully');
  } catch (e) {
    if (e.code === 23) {
      print('Replica set already initialized');
    } else {
      throw e;
    }
  }
"

echo "Waiting for replica set to elect primary..."
until mongosh "mongodb://$MONGO_USER:$MONGO_PASSWORD@mongo-primary:27017/admin" \
  --eval "rs.status().members.filter(m => m.stateStr === 'PRIMARY').length > 0" \
  --quiet | grep -q "true"; do
  echo "Waiting for primary election..."
  sleep 3
done

echo "Replica set initialized successfully!"

# Display replica set status
mongosh "mongodb://$MONGO_USER:$MONGO_PASSWORD@mongo-primary:27017/admin" --eval "rs.status()"
```

### Cluster Shardé avec Docker Compose

```yaml
# docker-compose-sharded.yml
version: '3.9'

x-mongo-base: &mongo-base
  image: mongo:7.0.5
  restart: unless-stopped
  networks:
    - mongodb_cluster
  logging:
    driver: "json-file"
    options:
      max-size: "50m"
      max-file: "3"

services:
  #############################
  # Config Server Replica Set #
  #############################

  config-server-1:
    <<: *mongo-base
    container_name: config-server-1
    hostname: config-server-1

    command: >
      mongod
      --configsvr
      --replSet configReplSet
      --bind_ip_all
      --port 27019

    ports:
      - "27019:27019"

    volumes:
      - config_server_1_data:/data/db

    healthcheck:
      test: ["CMD", "mongosh", "--port", "27019", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3

  config-server-2:
    <<: *mongo-base
    container_name: config-server-2
    hostname: config-server-2

    command: >
      mongod
      --configsvr
      --replSet configReplSet
      --bind_ip_all
      --port 27019

    ports:
      - "27020:27019"

    volumes:
      - config_server_2_data:/data/db

  config-server-3:
    <<: *mongo-base
    container_name: config-server-3
    hostname: config-server-3

    command: >
      mongod
      --configsvr
      --replSet configReplSet
      --bind_ip_all
      --port 27019

    ports:
      - "27021:27019"

    volumes:
      - config_server_3_data:/data/db

  #############################
  # Shard 0 Replica Set       #
  #############################

  shard0-node1:
    <<: *mongo-base
    container_name: shard0-node1
    hostname: shard0-node1

    command: >
      mongod
      --shardsvr
      --replSet shard0
      --bind_ip_all
      --port 27018

    ports:
      - "27030:27018"

    volumes:
      - shard0_node1_data:/data/db

  shard0-node2:
    <<: *mongo-base
    container_name: shard0-node2
    hostname: shard0-node2

    command: >
      mongod
      --shardsvr
      --replSet shard0
      --bind_ip_all
      --port 27018

    ports:
      - "27031:27018"

    volumes:
      - shard0_node2_data:/data/db

  shard0-node3:
    <<: *mongo-base
    container_name: shard0-node3
    hostname: shard0-node3

    command: >
      mongod
      --shardsvr
      --replSet shard0
      --bind_ip_all
      --port 27018

    ports:
      - "27032:27018"

    volumes:
      - shard0_node3_data:/data/db

  #############################
  # Shard 1 Replica Set       #
  #############################

  shard1-node1:
    <<: *mongo-base
    container_name: shard1-node1
    hostname: shard1-node1

    command: >
      mongod
      --shardsvr
      --replSet shard1
      --bind_ip_all
      --port 27018

    ports:
      - "27040:27018"

    volumes:
      - shard1_node1_data:/data/db

  shard1-node2:
    <<: *mongo-base
    container_name: shard1-node2
    hostname: shard1-node2

    command: >
      mongod
      --shardsvr
      --replSet shard1
      --bind_ip_all
      --port 27018

    ports:
      - "27041:27018"

    volumes:
      - shard1_node2_data:/data/db

  shard1-node3:
    <<: *mongo-base
    container_name: shard1-node3
    hostname: shard1-node3

    command: >
      mongod
      --shardsvr
      --replSet shard1
      --bind_ip_all
      --port 27018

    ports:
      - "27042:27018"

    volumes:
      - shard1_node3_data:/data/db

  #############################
  # Mongos Routers            #
  #############################

  mongos-1:
    <<: *mongo-base
    container_name: mongos-1
    hostname: mongos-1

    command: >
      mongos
      --configdb configReplSet/config-server-1:27019,config-server-2:27019,config-server-3:27019
      --bind_ip_all
      --port 27017

    ports:
      - "27017:27017"

    depends_on:
      - config-server-1
      - config-server-2
      - config-server-3

  mongos-2:
    <<: *mongo-base
    container_name: mongos-2
    hostname: mongos-2

    command: >
      mongos
      --configdb configReplSet/config-server-1:27019,config-server-2:27019,config-server-3:27019
      --bind_ip_all
      --port 27017

    ports:
      - "27018:27017"

    depends_on:
      - config-server-1
      - config-server-2
      - config-server-3

  #############################
  # Cluster Initializer       #
  #############################

  cluster-init:
    image: mongo:7.0.5
    container_name: cluster-init

    volumes:
      - ./scripts/init-sharded-cluster.sh:/init-cluster.sh:ro

    networks:
      - mongodb_cluster

    command: >
      bash -c "
        sleep 45 &&
        /init-cluster.sh
      "

    depends_on:
      - config-server-1
      - config-server-2
      - config-server-3
      - shard0-node1
      - shard0-node2
      - shard0-node3
      - shard1-node1
      - shard1-node2
      - shard1-node3
      - mongos-1

    restart: "no"

  #############################
  # Monitoring                #
  #############################

  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb-exporter
    restart: unless-stopped

    environment:
      MONGODB_URI: "mongodb://mongos-1:27017"

    ports:
      - "9216:9216"

    command:
      - "--collect-all"
      - "--compatible-mode"

    networks:
      - mongodb_cluster

    depends_on:
      - mongos-1

volumes:
  config_server_1_data:
  config_server_2_data:
  config_server_3_data:
  shard0_node1_data:
  shard0_node2_data:
  shard0_node3_data:
  shard1_node1_data:
  shard1_node2_data:
  shard1_node3_data:

networks:
  mongodb_cluster:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/16
```

```bash
#!/bin/bash
# scripts/init-sharded-cluster.sh

set -euo pipefail

echo "Initializing MongoDB Sharded Cluster..."

# Initialize Config Server Replica Set
echo "Initializing Config Server Replica Set..."
mongosh --host config-server-1 --port 27019 --eval "
  rs.initiate({
    _id: 'configReplSet',
    configsvr: true,
    members: [
      { _id: 0, host: 'config-server-1:27019' },
      { _id: 1, host: 'config-server-2:27019' },
      { _id: 2, host: 'config-server-3:27019' }
    ]
  })
"

sleep 10

# Initialize Shard 0 Replica Set
echo "Initializing Shard 0 Replica Set..."
mongosh --host shard0-node1 --port 27018 --eval "
  rs.initiate({
    _id: 'shard0',
    members: [
      { _id: 0, host: 'shard0-node1:27018' },
      { _id: 1, host: 'shard0-node2:27018' },
      { _id: 2, host: 'shard0-node3:27018' }
    ]
  })
"

# Initialize Shard 1 Replica Set
echo "Initializing Shard 1 Replica Set..."
mongosh --host shard1-node1 --port 27018 --eval "
  rs.initiate({
    _id: 'shard1',
    members: [
      { _id: 0, host: 'shard1-node1:27018' },
      { _id: 1, host: 'shard1-node2:27018' },
      { _id: 2, host: 'shard1-node3:27018' }
    ]
  })
"

sleep 20

# Add shards to the cluster
echo "Adding shards to the cluster..."
mongosh --host mongos-1 --port 27017 --eval "
  sh.addShard('shard0/shard0-node1:27018,shard0-node2:27018,shard0-node3:27018');
  sh.addShard('shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018');
"

sleep 5

# Display cluster status
echo "Cluster Status:"
mongosh --host mongos-1 --port 27017 --eval "sh.status()"

echo "Sharded Cluster initialized successfully!"
```

---

## Gestion des Volumes et Persistance

### Drivers de Volume Docker

**1. Local Driver (Default)**

```yaml
volumes:
  mongodb_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/data/mongodb
```

**Avantages :**
- Simple à configurer
- Performance native du système de fichiers
- Pas de dépendances externes

**Inconvénients :**
- Lié au host
- Pas de réplication automatique
- Migration manuelle

**2. Volume Driver NFS**

```yaml
volumes:
  mongodb_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server.example.com,rw,nfsvers=4.1
      device: ":/exports/mongodb"
```

**3. Volume Driver Cloud (AWS EBS, GCE PD)**

```yaml
# Docker plugin pour AWS EBS
docker plugin install rexray/ebs \
  EBS_ACCESSKEY=xxx \
  EBS_SECRETKEY=xxx \
  EBS_REGION=eu-west-3

volumes:
  mongodb_data:
    driver: rexray/ebs
    driver_opts:
      size: 500
      volumetype: gp3
      iops: 16000
```

### Stratégies de Backup avec Docker

```yaml
# docker-compose-backup.yml
version: '3.9'

services:
  mongodb-backup:
    image: mongo:7.0.5
    container_name: mongodb-backup

    environment:
      MONGODB_URI: "mongodb://admin:${MONGO_ROOT_PASSWORD}@mongo-primary:27017/?replicaSet=rs0"
      BACKUP_SCHEDULE: "0 2 * * *"  # Daily at 2 AM
      BACKUP_RETENTION_DAYS: 30
      S3_BUCKET: "s3://company-mongodb-backups"
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}

    volumes:
      - ./scripts/backup-mongodb.sh:/usr/local/bin/backup.sh:ro
      - backup_storage:/backup
      - /var/run/docker.sock:/var/run/docker.sock:ro

    command: >
      bash -c "
        apt-get update &&
        apt-get install -y cron awscli &&
        echo \"$$BACKUP_SCHEDULE root /usr/local/bin/backup.sh\" > /etc/cron.d/mongodb-backup &&
        chmod 0644 /etc/cron.d/mongodb-backup &&
        cron -f
      "

    networks:
      - mongodb_network

    depends_on:
      - mongo-primary

volumes:
  backup_storage:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/backups/mongodb

networks:
  mongodb_network:
    external: true
```

```bash
#!/bin/bash
# scripts/backup-mongodb.sh

set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/$TIMESTAMP"
MONGODB_URI="${MONGODB_URI}"
S3_BUCKET="${S3_BUCKET}"
RETENTION_DAYS="${BACKUP_RETENTION_DAYS:-30}"

echo "Starting MongoDB backup: $TIMESTAMP"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Run mongodump
mongodump \
  --uri="$MONGODB_URI" \
  --out="$BACKUP_DIR" \
  --gzip \
  --oplog

# Compress backup
cd /backup
tar czf "${TIMESTAMP}.tar.gz" "$TIMESTAMP"

# Calculate size
BACKUP_SIZE=$(du -h "${TIMESTAMP}.tar.gz" | cut -f1)
echo "Backup size: $BACKUP_SIZE"

# Upload to S3
if [ -n "$S3_BUCKET" ]; then
  echo "Uploading to S3..."
  aws s3 cp "${TIMESTAMP}.tar.gz" "$S3_BUCKET/${TIMESTAMP}.tar.gz" \
    --storage-class STANDARD_IA

  # Verify upload
  if aws s3 ls "$S3_BUCKET/${TIMESTAMP}.tar.gz" > /dev/null 2>&1; then
    echo "Backup uploaded successfully to S3"
    # Remove local backup after successful upload
    rm -rf "$BACKUP_DIR" "${TIMESTAMP}.tar.gz"
  else
    echo "Failed to upload backup to S3"
    exit 1
  fi
fi

# Cleanup old backups from S3
echo "Cleaning up old backups (older than $RETENTION_DAYS days)..."
aws s3 ls "$S3_BUCKET/" | \
  awk '{print $4}' | \
  while read -r backup_file; do
    backup_date=$(echo "$backup_file" | grep -oP '\d{8}')
    if [ -n "$backup_date" ]; then
      days_old=$(( ($(date +%s) - $(date -d "$backup_date" +%s)) / 86400 ))
      if [ "$days_old" -gt "$RETENTION_DAYS" ]; then
        echo "Deleting old backup: $backup_file (${days_old} days old)"
        aws s3 rm "$S3_BUCKET/$backup_file"
      fi
    fi
  done

echo "Backup completed successfully!"

# Send notification (optional)
if [ -n "${SLACK_WEBHOOK_URL:-}" ]; then
  curl -X POST "$SLACK_WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "{\"text\":\"MongoDB backup completed: $TIMESTAMP (Size: $BACKUP_SIZE)\"}"
fi
```

---

## Networking Docker pour MongoDB

### Network Modes

**1. Bridge Network (Default)**

```yaml
networks:
  mongodb_network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
```

**Caractéristiques :**
- Isolation réseau entre conteneurs
- Service discovery via DNS
- NAT pour accès externe

**2. Host Network**

```yaml
services:
  mongodb:
    network_mode: host
```

**Avantages :**
- Performance maximale (pas de NAT)
- Latence minimale

**Inconvénients :**
- Pas d'isolation réseau
- Conflits de ports possibles
- Moins sécurisé

**3. Overlay Network (Docker Swarm)**

```yaml
networks:
  mongodb_cluster:
    driver: overlay
    attachable: true
    driver_opts:
      encrypted: "true"
    ipam:
      config:
        - subnet: 10.10.0.0/16
```

**Avantages :**
- Multi-host networking
- Chiffrement natif
- Service discovery distribué

### Configuration DNS pour Replica Set

```yaml
# docker-compose.yml avec DNS personnalisé
version: '3.9'

services:
  mongo-primary:
    image: mongo:7.0.5
    hostname: mongo-primary.local
    domainname: mongodb.local

    networks:
      mongodb_network:
        aliases:
          - mongo-primary.mongodb.local
          - primary.mongodb.local

    dns:
      - 8.8.8.8
      - 8.8.4.4

    dns_search:
      - mongodb.local

    extra_hosts:
      - "mongo-secondary-1.local:172.20.0.3"
      - "mongo-secondary-2.local:172.20.0.4"

networks:
  mongodb_network:
    driver: bridge
    enable_ipv6: false
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Service Discovery avec Consul

```yaml
# docker-compose-service-discovery.yml
version: '3.9'

services:
  consul:
    image: consul:1.16
    container_name: consul

    command: agent -server -bootstrap -ui -client=0.0.0.0

    ports:
      - "8500:8500"
      - "8600:8600/udp"

    networks:
      - mongodb_network

  registrator:
    image: gliderlabs/registrator:latest
    container_name: registrator

    command: -internal consul://consul:8500

    volumes:
      - /var/run/docker.sock:/tmp/docker.sock

    networks:
      - mongodb_network

    depends_on:
      - consul

  mongo-primary:
    image: mongo:7.0.5

    environment:
      SERVICE_NAME: mongodb-primary
      SERVICE_TAGS: mongodb,primary,replica-set
      SERVICE_CHECK_SCRIPT: "mongosh --eval \"db.adminCommand('ping')\""
      SERVICE_CHECK_INTERVAL: 30s

    networks:
      - mongodb_network

    depends_on:
      - registrator

networks:
  mongodb_network:
    driver: bridge
```

---

## Sécurité Docker pour MongoDB

### Secrets Management

```yaml
# docker-compose-secrets.yml
version: '3.9'

secrets:
  mongo_root_password:
    external: true

  mongo_keyfile:
    external: true

  mongo_tls_cert:
    external: true

  mongo_tls_key:
    external: true

services:
  mongodb:
    image: mongo:7.0.5

    secrets:
      - source: mongo_root_password
        target: /run/secrets/mongo_root_password
        mode: 0400

      - source: mongo_keyfile
        target: /run/secrets/keyfile
        mode: 0400

      - source: mongo_tls_cert
        target: /run/secrets/tls_cert
        mode: 0444

      - source: mongo_tls_key
        target: /run/secrets/tls_key
        mode: 0400

    environment:
      MONGO_INITDB_ROOT_PASSWORD_FILE: /run/secrets/mongo_root_password

    command: >
      mongod
      --keyFile /run/secrets/keyfile
      --tlsMode requireTLS
      --tlsCertificateKeyFile /run/secrets/tls_cert
      --tlsCAFile /run/secrets/ca_cert
```

```bash
# Create Docker secrets
echo "supersecretpassword" | docker secret create mongo_root_password -

openssl rand -base64 756 | docker secret create mongo_keyfile -

docker secret create mongo_tls_cert mongodb.pem
docker secret create mongo_tls_key mongodb-key.pem
```

### AppArmor Profile

```bash
# /etc/apparmor.d/docker-mongodb
#include <tunables/global>

profile docker-mongodb flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # Allow network operations
  network inet stream,
  network inet6 stream,

  # Allow reading from specific directories
  /data/db/** rw,
  /data/configdb/** rw,
  /var/log/mongodb/** rw,

  # Allow execution of MongoDB binaries
  /usr/bin/mongod ix,
  /usr/bin/mongosh ix,

  # Deny access to sensitive areas
  deny /proc/** w,
  deny /sys/** w,
  deny /root/** rw,

  # Allow reading configuration
  /etc/mongod.conf r,
  /etc/mongodb/** r,
}
```

```yaml
# docker-compose.yml with AppArmor
services:
  mongodb:
    image: mongo:7.0.5
    security_opt:
      - apparmor=docker-mongodb
      - no-new-privileges:true

    cap_drop:
      - ALL

    cap_add:
      - CHOWN
      - SETUID
      - SETGID
      - DAC_OVERRIDE

    read_only: true

    tmpfs:
      - /tmp
      - /run
```

### User Namespace Remapping

```json
// /etc/docker/daemon.json
{
  "userns-remap": "mongodb",
  "storage-driver": "overlay2",
  "live-restore": true,
  "userland-proxy": false
}
```

```bash
# Create user namespace mapping
sudo useradd -r -s /bin/false mongodb

# Configure subuid and subgid
echo "mongodb:100000:65536" | sudo tee -a /etc/subuid
echo "mongodb:100000:65536" | sudo tee -a /etc/subgid

# Restart Docker
sudo systemctl restart docker
```

---

## Monitoring et Logging

### Prometheus + Grafana Stack

```yaml
# docker-compose-monitoring.yml
version: '3.9'

services:
  # MongoDB instances (from previous configs)
  # ...

  prometheus:
    image: prom/prometheus:v2.45.0
    container_name: prometheus

    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus

    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'

    ports:
      - "9090:9090"

    networks:
      - mongodb_network

    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.0.3
    container_name: grafana

    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
      GF_INSTALL_PLUGINS: grafana-mongodb-datasource

    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
      - grafana_data:/var/lib/grafana

    ports:
      - "3000:3000"

    networks:
      - mongodb_network

    depends_on:
      - prometheus

    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager

    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager

    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'

    ports:
      - "9093:9093"

    networks:
      - mongodb_network

    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter

    command:
      - '--path.rootfs=/host'

    volumes:
      - /:/host:ro,rslave

    ports:
      - "9100:9100"

    networks:
      - mongodb_network

    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

networks:
  mongodb_network:
    external: true
```

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 30s
  evaluation_interval: 30s
  external_labels:
    cluster: 'mongodb-docker'
    environment: 'production'

rule_files:
  - '/etc/prometheus/rules/*.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets:
          - 'mongodb-exporter:9216'

    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+)(?::\d+)?'
        replacement: '$1'

  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-exporter:9100'

  - job_name: 'prometheus'
    static_configs:
      - targets:
          - 'localhost:9090'
```

```yaml
# prometheus/rules/mongodb-alerts.yml
groups:
  - name: mongodb_alerts
    interval: 30s
    rules:
      - alert: MongoDBDown
        expr: up{job="mongodb"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB instance down"
          description: "MongoDB instance {{ $labels.instance }} is down"

      - alert: MongoDBHighConnections
        expr: mongodb_connections{state="current"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB high connections"
          description: "MongoDB has {{ $value }} connections on {{ $labels.instance }}"

      - alert: MongoDBReplicationLag
        expr: mongodb_mongod_replset_member_replication_lag > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB replication lag"
          description: "Replication lag is {{ $value }}s on {{ $labels.instance }}"

      - alert: MongoDBHighCPU
        expr: rate(mongodb_sys_cpu_num_cpus[5m]) > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB high CPU usage"
          description: "CPU usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"
```

### Logging Centralise avec ELK

```yaml
# docker-compose-logging.yml
version: '3.9'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.9.0
    container_name: elasticsearch

    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"

    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

    ports:
      - "9200:9200"

    networks:
      - logging_network

    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

  logstash:
    image: docker.elastic.co/logstash/logstash:8.9.0
    container_name: logstash

    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/patterns:/usr/share/logstash/patterns:ro

    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"

    environment:
      LS_JAVA_OPTS: "-Xmx1g -Xms1g"

    networks:
      - logging_network

    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.9.0
    container_name: kibana

    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200

    ports:
      - "5601:5601"

    networks:
      - logging_network

    depends_on:
      elasticsearch:
        condition: service_healthy

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.9.0
    container_name: filebeat
    user: root

    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - filebeat_data:/usr/share/filebeat/data

    command: filebeat -e -strict.perms=false

    networks:
      - logging_network
      - mongodb_network

    depends_on:
      - logstash

volumes:
  elasticsearch_data:
  filebeat_data:

networks:
  logging_network:
    driver: bridge
  mongodb_network:
    external: true
```

```yaml
# logstash/pipeline/mongodb.conf
input {
  beats {
    port => 5044
  }

  tcp {
    port => 5000
    codec => json
  }
}

filter {
  if [container][name] =~ "mongo" {
    # Parse MongoDB logs
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{WORD:severity} %{WORD:component} *\[%{DATA:context}\] %{GREEDYDATA:message}"
      }
      overwrite => ["message"]
    }

    # Parse MongoDB slow queries
    if [message] =~ "COMMAND" or [message] =~ "WRITE" {
      grok {
        match => {
          "message" => "(%{NUMBER:duration}ms)"
        }
      }

      if [duration] {
        mutate {
          convert => { "duration" => "integer" }
        }
      }
    }

    # Add custom fields
    mutate {
      add_field => {
        "[@metadata][index_prefix]" => "mongodb-logs"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][index_prefix]}-%{+YYYY.MM.dd}"
  }

  # Debug output
  stdout {
    codec => rubydebug
  }
}
```

---

## CI/CD avec Docker

### Jenkinsfile pour MongoDB

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io/company'
        IMAGE_NAME = 'mongodb'
        MONGO_VERSION = '7.0.5'
        COMPOSE_PROJECT_NAME = 'mongodb-ci'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Custom Image') {
            steps {
                script {
                    docker.build(
                        "${DOCKER_REGISTRY}/${IMAGE_NAME}:${MONGO_VERSION}",
                        "-f Dockerfile.mongodb ."
                    )
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    sh """
                        docker scan ${DOCKER_REGISTRY}/${IMAGE_NAME}:${MONGO_VERSION} \
                            --severity high \
                            --json > scan-results.json || true
                    """

                    // Parse results and fail if critical vulnerabilities found
                    def scanResults = readJSON file: 'scan-results.json'
                    if (scanResults.vulnerabilities.critical > 0) {
                        error("Critical vulnerabilities found!")
                    }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                script {
                    sh '''
                        # Start test MongoDB instance
                        docker-compose -f docker-compose-test.yml up -d

                        # Wait for MongoDB to be ready
                        sleep 20

                        # Run tests
                        docker-compose -f docker-compose-test.yml exec -T mongodb \
                            mongosh --eval "db.adminCommand('ping')"

                        # Run application tests
                        docker run --rm --network=${COMPOSE_PROJECT_NAME}_default \
                            -e MONGODB_URI="mongodb://mongodb:27017/test" \
                            company/app-tests:latest
                    '''
                }
            }
        }

        stage('Integration Tests') {
            steps {
                script {
                    sh '''
                        # Deploy replica set
                        docker-compose -f docker-compose-replica-set.yml up -d

                        # Wait for replica set initialization
                        sleep 60

                        # Run integration tests
                        docker-compose -f docker-compose-replica-set.yml \
                            exec -T mongo-primary \
                            mongosh --eval "
                                rs.status().ok === 1 || quit(1);
                                rs.status().members.filter(m => m.stateStr === 'PRIMARY').length === 1 || quit(1);
                            "
                    '''
                }
            }
        }

        stage('Performance Tests') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                        # Run load tests
                        docker run --rm --network=${COMPOSE_PROJECT_NAME}_default \
                            -v $(pwd)/tests:/tests \
                            grafana/k6 run /tests/load-test.js
                    '''
                }
            }
        }

        stage('Push Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://docker.io', 'docker-credentials') {
                        def image = docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${MONGO_VERSION}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                        # Deploy to staging environment
                        ssh staging-server << 'EOF'
                            cd /opt/mongodb
                            docker-compose pull
                            docker-compose up -d --no-deps --build

                            # Health check
                            sleep 30
                            docker-compose exec -T mongodb mongosh --eval "db.adminCommand('ping')"
                        EOF
                    '''
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                script {
                    sh '''
                        # Blue-Green deployment
                        ansible-playbook -i inventory/production \
                            playbooks/deploy-mongodb.yml \
                            --extra-vars "version=${MONGO_VERSION}"
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                sh '''
                    # Cleanup
                    docker-compose -f docker-compose-test.yml down -v || true
                    docker-compose -f docker-compose-replica-set.yml down -v || true
                '''
            }
        }

        success {
            slackSend(
                color: 'good',
                message: "MongoDB deployment successful: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }

        failure {
            slackSend(
                color: 'danger',
                message: "MongoDB deployment failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }
    }
}
```

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  LATEST_TAG: $CI_REGISTRY_IMAGE:latest

stages:
  - build
  - test
  - security
  - deploy

.docker_login: &docker_login
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  <<: *docker_login
  script:
    - docker build -t $IMAGE_TAG -f Dockerfile.mongodb .
    - docker push $IMAGE_TAG
  only:
    - branches
  tags:
    - docker

test:unit:
  stage: test
  services:
    - name: mongo:7.0.5
      alias: mongodb
  variables:
    MONGODB_URI: "mongodb://mongodb:27017/test"
  script:
    - apt-get update && apt-get install -y mongodb-mongosh
    - mongosh $MONGODB_URI --eval "db.adminCommand('ping')"
    - npm test
  only:
    - branches
  tags:
    - docker

test:replica-set:
  stage: test
  script:
    - docker-compose -f docker-compose-replica-set.yml up -d
    - sleep 60
    - docker-compose -f docker-compose-replica-set.yml exec -T mongo-primary mongosh --eval "rs.status()"
    - docker-compose -f docker-compose-replica-set.yml down -v
  only:
    - branches
  tags:
    - docker

security:scan:
  stage: security
  script:
    - docker pull $IMAGE_TAG
    - docker scan $IMAGE_TAG --file Dockerfile.mongodb --json > scan-results.json
    - cat scan-results.json | jq '.vulnerabilities[] | select(.severity=="high" or .severity=="critical")'
  artifacts:
    reports:
      container_scanning: scan-results.json
    expire_in: 1 week
  only:
    - branches
  tags:
    - docker

deploy:staging:
  stage: deploy
  <<: *docker_login
  script:
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $LATEST_TAG
    - docker push $LATEST_TAG

    # Deploy to staging
    - |
      ssh staging-server << EOF
        cd /opt/mongodb
        docker-compose pull
        docker-compose up -d --no-deps mongodb
      EOF
  environment:
    name: staging
    url: https://mongodb-staging.company.com
  only:
    - develop
  tags:
    - docker

deploy:production:
  stage: deploy
  <<: *docker_login
  script:
    - docker pull $IMAGE_TAG

    # Deploy to production with rolling update
    - |
      for host in prod-mongo-{1..3}; do
        ssh $host << EOF
          cd /opt/mongodb
          docker pull $IMAGE_TAG
          docker-compose up -d --no-deps mongodb
          sleep 60
        EOF
      done
  environment:
    name: production
    url: https://mongodb.company.com
  when: manual
  only:
    - main
  tags:
    - docker
```

---

## Production Best Practices

### Resource Limits et QoS

```yaml
services:
  mongodb:
    image: mongo:7.0.5

    # CPU limits
    cpus: '4.0'
    cpu_shares: 1024

    # Memory limits
    mem_limit: 16g
    mem_reservation: 8g
    memswap_limit: 16g

    # I/O limits
    blkio_config:
      weight: 500
      device_read_bps:
        - path: /dev/sda
          rate: '50mb'
      device_write_bps:
        - path: /dev/sda
          rate: '50mb'

    # PID limits
    pids_limit: 4096

    # OOM adjustments
    oom_score_adj: -500

    deploy:
      mode: replicated
      replicas: 1

      resources:
        limits:
          cpus: '4'
          memory: 16G
        reservations:
          cpus: '2'
          memory: 8G

      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s

      placement:
        constraints:
          - node.role == worker
          - node.labels.storage == ssd
```

### Optimisations Performance

```bash
# Tuning du host pour MongoDB
cat > /etc/sysctl.d/99-mongodb.conf << 'EOF'
# Network tuning
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.core.netdev_max_backlog = 5000

# File descriptors
fs.file-max = 98000

# Virtual memory
vm.swappiness = 1
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# Transparent Huge Pages
transparent_hugepage=never
EOF

sysctl -p /etc/sysctl.d/99-mongodb.conf
```

### Checklist Production

```yaml
# production-checklist.yml
---
infrastructure:
  - ✅ Volumes persistants configurés avec driver approprié
  - ✅ Backups automatiques configurés
  - ✅ Monitoring (Prometheus + Grafana) déployé
  - ✅ Logging centralisé (ELK) configuré
  - ✅ Alerting configuré (PagerDuty, Slack)

security:
  - ✅ Authentication activée
  - ✅ TLS/SSL configuré
  - ✅ Keyfile pour replica set
  - ✅ Secrets management avec Docker Secrets
  - ✅ AppArmor/SELinux profiles appliqués
  - ✅ User namespace remapping activé
  - ✅ Audit logging activé

networking:
  - ✅ Network isolation configurée
  - ✅ Service discovery fonctionnel
  - ✅ DNS resolution stable
  - ✅ Firewall rules configurées

performance:
  - ✅ Resource limits appropriés
  - ✅ WiredTiger cache configuré
  - ✅ Indexes optimisés
  - ✅ Connection pooling configuré
  - ✅ I/O performance testée

high_availability:
  - ✅ Replica Set avec 3+ membres
  - ✅ Anti-affinity rules configurées
  - ✅ Health checks configurés
  - ✅ Failover testé
  - ✅ Disaster recovery plan documenté

operational:
  - ✅ Documentation à jour
  - ✅ Runbooks disponibles
  - ✅ On-call rotation définie
  - ✅ Escalation path clair
  - ✅ Post-mortem process en place
```

---

## Conclusion

Le déploiement de MongoDB avec Docker offre :
- **Portabilité** : Déploiement cohérent sur tous les environnements
- **Isolation** : Conteneurisation des dépendances et configurations
- **Rapidité** : Déploiement et scaling rapides
- **Automatisation** : CI/CD simplifié

Pour la production, privilégier :
- **Kubernetes** pour orchestration avancée
- **Volumes managés** pour performance I/O
- **Monitoring robuste** avec alerting
- **Backups automatisés** avec rétention appropriée

---


⏭️ [Docker Compose pour environnements de dev](/18-devops-deploiement/03-docker-compose-dev.md)
