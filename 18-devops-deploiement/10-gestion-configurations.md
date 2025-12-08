🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.10 Gestion des Configurations

## Introduction

La gestion des configurations est un pilier fondamental du DevOps moderne. Elle permet de séparer la configuration du code, de gérer les environnements multiples, et de maintenir la sécurité tout en facilitant les déploiements. Pour les applications MongoDB, cela inclut la gestion des connection strings, des paramètres de performance, et des secrets sensibles.

```
┌────────────────────────────────────────────────────────────────────┐
│           Configuration Management Architecture                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                Configuration Sources                         │  │
│  │                                                              │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │   Git Repo   │  │    Vault     │  │   Consul     │        │  │
│  │  │              │  │              │  │              │        │  │
│  │  │  config/     │  │  • Secrets   │  │  • KV Store  │        │  │
│  │  │  ├─ dev.yml  │  │  • Tokens    │  │  • Service   │        │  │
│  │  │  ├─ prod.yml │  │  • Certs     │  │    Discovery │        │  │
│  │  │  └─ base.yml │  │              │  │              │        │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │  │
│  └────────────┬───────────────┬──────────────┬──────────────────┘  │
│               │               │              │                     │
│               └───────────────┴──────────────┘                     │
│                               │                                    │
│                               ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │            Configuration Management Layer                    │  │
│  │                                                              │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  Kubernetes Configuration                              │  │  │
│  │  │                                                        │  │  │
│  │  │  • ConfigMaps    - Non-sensitive configuration         │  │  │
│  │  │  • Secrets       - Sensitive data (base64)             │  │  │
│  │  │  • External      - Vault integration                   │  │  │
│  │  │    Secrets                                             │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                        │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Applications                              │  │
│  │                                                              │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │  │
│  │  │   API    │  │ Worker   │  │  Cron    │  │  Queue   │      │  │
│  │  │  Server  │  │          │  │  Jobs    │  │  Proc    │      │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘      │  │
│  │       │             │             │             │            │  │
│  │       └─────────────┴─────────────┴─────────────┘            │  │
│  │                           │                                  │  │
│  │                           ▼                                  │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │           MongoDB Cluster                              │  │  │
│  │  │                                                        │  │  │
│  │  │  • Connection String (from Secret)                     │  │  │
│  │  │  • TLS Certificates (from Secret)                      │  │  │
│  │  │  • Performance Config (from ConfigMap)                 │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Principes de la Configuration

```yaml
# configuration-principles.yaml
---
separation_of_concerns:
  principle: "Séparer code et configuration"
  benefits:
    - "Même code pour tous les environnements"
    - "Déploiements simplifiés"
    - "Pas de rebuild pour changer config"
    - "Audit trail des changements"

environment_parity:
  principle: "Parité entre environnements"
  benefits:
    - "Dev/Staging/Prod similaires"
    - "Bugs détectés plus tôt"
    - "Confiance dans les déploiements"
    - "Configuration testable"

security_by_default:
  principle: "Sécurité par défaut"
  benefits:
    - "Secrets jamais en clair"
    - "Encryption at rest"
    - "Access control"
    - "Audit logging"

immutability:
  principle: "Configuration immuable"
  benefits:
    - "Reproductibilité"
    - "Rollback facile"
    - "No configuration drift"
    - "GitOps workflows"

validation:
  principle: "Validation automatique"
  benefits:
    - "Erreurs détectées tôt"
    - "Schema enforcement"
    - "Type safety"
    - "Automated testing"
```

---

## ConfigMaps Kubernetes

### ConfigMap Basique

```yaml
# mongodb-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-config
  namespace: production
  labels:
    app: mongodb
    component: config
data:
  # Configuration MongoDB
  mongodb.conf: |
    # Network
    net:
      port: 27017
      bindIp: 0.0.0.0
      maxIncomingConnections: 65536

      # TLS/SSL
      tls:
        mode: requireTLS
        certificateKeyFile: /etc/mongodb/tls/mongodb.pem
        CAFile: /etc/mongodb/tls/ca.crt

    # Storage
    storage:
      dbPath: /data/db
      directoryPerDB: true
      engine: wiredTiger
      wiredTiger:
        engineConfig:
          cacheSizeGB: 8
          journalCompressor: snappy
          directoryForIndexes: true
        collectionConfig:
          blockCompressor: snappy
        indexConfig:
          prefixCompression: true

    # Replication
    replication:
      replSetName: rs0
      oplogSizeMB: 10240

    # Security
    security:
      authorization: enabled
      keyFile: /etc/mongodb/keyfile/mongodb-keyfile

    # Operation Profiling
    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 100

    # System Log
    systemLog:
      destination: file
      path: /var/log/mongodb/mongod.log
      logAppend: true
      verbosity: 0

  # Application Configuration
  app-config.json: |
    {
      "database": {
        "name": "myapp",
        "connectionTimeout": 30000,
        "socketTimeout": 30000,
        "serverSelectionTimeout": 30000,
        "maxPoolSize": 100,
        "minPoolSize": 10,
        "maxIdleTimeMS": 60000,
        "waitQueueTimeoutMS": 10000,
        "retryWrites": true,
        "retryReads": true
      },
      "cache": {
        "enabled": true,
        "ttl": 300,
        "maxSize": 1000
      },
      "logging": {
        "level": "info",
        "format": "json",
        "includeTimestamp": true
      },
      "performance": {
        "enableQueryLogging": true,
        "slowQueryThreshold": 100,
        "enableMetrics": true
      }
    }

  # Environment-specific settings
  MONGODB_DATABASE: "myapp"
  LOG_LEVEL: "info"
  CACHE_ENABLED: "true"
  CACHE_TTL: "300"
  MAX_POOL_SIZE: "100"
  MIN_POOL_SIZE: "10"
---
# ConfigMap pour scripts
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-scripts
  namespace: production
data:
  init-replica-set.sh: |
    #!/bin/bash
    set -e

    # Wait for MongoDB to be ready
    echo "Waiting for MongoDB..."
    until mongosh --eval "db.adminCommand('ping')" > /dev/null 2>&1; do
      sleep 2
    done

    echo "MongoDB is ready"

    # Check if replica set is already initialized
    STATUS=$(mongosh --eval "rs.status().ok" --quiet)

    if [ "$STATUS" = "1" ]; then
      echo "Replica set already initialized"
      exit 0
    fi

    echo "Initializing replica set..."

    # Initialize replica set
    mongosh <<EOF
    rs.initiate({
      _id: "rs0",
      members: [
        { _id: 0, host: "mongodb-0.mongodb-headless:27017", priority: 2 },
        { _id: 1, host: "mongodb-1.mongodb-headless:27017", priority: 1 },
        { _id: 2, host: "mongodb-2.mongodb-headless:27017", priority: 1 }
      ]
    })
    EOF

    echo "Replica set initialized"

  healthcheck.sh: |
    #!/bin/bash

    # Check if MongoDB is responding
    if ! mongosh --eval "db.adminCommand('ping')" > /dev/null 2>&1; then
      echo "MongoDB is not responding"
      exit 1
    fi

    # Check replica set status
    STATUS=$(mongosh --eval "JSON.stringify(rs.status())" --quiet)

    # Check if this node is in the replica set
    if echo "$STATUS" | jq -e '.ok == 1' > /dev/null; then
      echo "Node is healthy"
      exit 0
    else
      echo "Node is not healthy"
      exit 1
    fi
```

### ConfigMap Multi-Environnement

```yaml
# configmap-dev.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
  labels:
    app: myapp
    environment: development
data:
  NODE_ENV: "development"
  LOG_LEVEL: "debug"
  CACHE_ENABLED: "false"
  MONGODB_DATABASE: "myapp_dev"
  MAX_POOL_SIZE: "10"
  ENABLE_PROFILING: "true"
  FEATURE_NEW_API: "true"
---
# configmap-staging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: staging
  labels:
    app: myapp
    environment: staging
data:
  NODE_ENV: "staging"
  LOG_LEVEL: "info"
  CACHE_ENABLED: "true"
  CACHE_TTL: "300"
  MONGODB_DATABASE: "myapp_staging"
  MAX_POOL_SIZE: "50"
  ENABLE_PROFILING: "true"
  FEATURE_NEW_API: "true"
---
# configmap-production.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: myapp
    environment: production
data:
  NODE_ENV: "production"
  LOG_LEVEL: "warn"
  CACHE_ENABLED: "true"
  CACHE_TTL: "600"
  MONGODB_DATABASE: "myapp"
  MAX_POOL_SIZE: "100"
  MIN_POOL_SIZE: "20"
  ENABLE_PROFILING: "false"
  FEATURE_NEW_API: "false"
```

### Utilisation dans un Deployment

```yaml
# deployment-with-configmap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        # Restart pods when ConfigMap changes
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0

          # Environment variables from ConfigMap
          envFrom:
            - configMapRef:
                name: app-config

          # Specific environment variables
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: connection-string

            # Override from ConfigMap
            - name: DATABASE_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: MONGODB_DATABASE

          # Mount config files
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true

            - name: scripts
              mountPath: /app/scripts
              readOnly: true

      volumes:
        # ConfigMap as volume
        - name: config
          configMap:
            name: mongodb-config
            items:
              - key: app-config.json
                path: config.json

        - name: scripts
          configMap:
            name: mongodb-scripts
            defaultMode: 0755
```

---

## Secrets Kubernetes

### Secrets Basiques

```yaml
# mongodb-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: production
  labels:
    app: mongodb
type: Opaque
stringData:
  # Root credentials
  root-username: admin
  root-password: SuperSecurePassword123!

  # Application user
  app-username: appuser
  app-password: AppPassword456!

  # Connection strings
  connection-string: mongodb://appuser:AppPassword456!@mongodb-0.mongodb-headless:27017,mongodb-1.mongodb-headless:27017,mongodb-2.mongodb-headless:27017/myapp?replicaSet=rs0&authSource=admin

  connection-string-srv: mongodb+srv://appuser:AppPassword456!@mongodb-cluster.example.com/myapp?retryWrites=true&w=majority

  # Replica set keyfile
  replica-set-key: |
    aWFLTnNvdVJzY01MQjRnNklHT1RIZERqQnNJZ3ZzNHlvSEZoZUZQTGJEQWpG
    UXhGS0dmMzFRZ3hMUG9LR29vQ2dZRUFtTlZxWnROUzRzZGpxUjJaMlVHMGpv
---
# TLS Certificates Secret
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-tls
  namespace: production
  labels:
    app: mongodb
type: kubernetes.io/tls
data:
  # Base64 encoded certificate
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0t...
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
---
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJteXJlZ2lzdHJ5LmNvbSI6eyJ1c2VybmFtZSI6InVzZXIiLCJwYXNzd29yZCI6InBhc3MiLCJlbWFpbCI6ImVtYWlsQGV4YW1wbGUuY29tIiwiYXV0aCI6ImRYTmxjanB3WVhOeiJ9fX0=
```

### Secrets via Kustomize

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

# Generate secrets from files
secretGenerator:
  - name: mongodb-credentials
    literals:
      - username=admin
      - password=SuperSecurePassword123!
    options:
      disableNameSuffixHash: false

  - name: mongodb-tls-files
    files:
      - tls/mongodb.crt
      - tls/mongodb.key
      - tls/ca.crt
    type: kubernetes.io/tls

  - name: mongodb-keyfile
    files:
      - keyfile=mongodb-keyfile.txt
    options:
      disableNameSuffixHash: true

# Generate ConfigMaps
configMapGenerator:
  - name: mongodb-config
    files:
      - config/mongodb.conf
      - config/app-config.json
    options:
      disableNameSuffixHash: false

resources:
  - deployment.yaml
  - service.yaml
```

---

## HashiCorp Vault Integration

### Vault Configuration

```hcl
# vault-config.hcl
# MongoDB Secrets Engine configuration

# Enable MongoDB secrets engine
path "sys/mounts/mongodb" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Configure MongoDB connection
path "mongodb/config/myapp" {
  capabilities = ["create", "read", "update", "delete"]
}

# Define roles
path "mongodb/roles/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Allow reading credentials
path "mongodb/creds/*" {
  capabilities = ["read"]
}

# KV secrets for static config
path "secret/data/mongodb/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# PKI for certificates
path "pki/issue/mongodb" {
  capabilities = ["create", "update"]
}
```

### Vault CLI Setup

```bash
#!/bin/bash
# scripts/vault-setup.sh
# Setup Vault for MongoDB

set -e

# Enable secrets engines
vault secrets enable -path=mongodb database
vault secrets enable -version=2 -path=secret kv

# Configure MongoDB connection
vault write mongodb/config/myapp \
  plugin_name=mongodb-database-plugin \
  allowed_roles="app-role,readonly-role" \
  connection_url="mongodb://{{username}}:{{password}}@mongodb-0.mongodb-headless:27017,mongodb-1.mongodb-headless:27017,mongodb-2.mongodb-headless:27017/admin?replicaSet=rs0" \
  username="vaultadmin" \
  password="VaultAdminPassword123!"

# Create application role
vault write mongodb/roles/app-role \
  db_name=myapp \
  creation_statements='{"db": "myapp", "roles": [{"role": "readWrite"}]}' \
  default_ttl="1h" \
  max_ttl="24h"

# Create read-only role
vault write mongodb/roles/readonly-role \
  db_name=myapp \
  creation_statements='{"db": "myapp", "roles": [{"role": "read"}]}' \
  default_ttl="1h" \
  max_ttl="24h"

# Store static configuration
vault kv put secret/mongodb/config \
  database_name="myapp" \
  max_pool_size="100" \
  min_pool_size="10" \
  cache_ttl="300"

# Store connection parameters
vault kv put secret/mongodb/connection \
  replica_set="rs0" \
  auth_source="admin" \
  retry_writes="true" \
  w="majority"

# Enable PKI for certificates
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

# Generate root CA
vault write pki/root/generate/internal \
  common_name="MongoDB Internal CA" \
  ttl=87600h

# Configure CA and CRL URLs
vault write pki/config/urls \
  issuing_certificates="http://vault:8200/v1/pki/ca" \
  crl_distribution_points="http://vault:8200/v1/pki/crl"

# Create role for MongoDB certificates
vault write pki/roles/mongodb \
  allowed_domains="mongodb,mongodb-headless,*.mongodb-headless.production.svc.cluster.local" \
  allow_subdomains=true \
  max_ttl="720h"

echo "Vault setup completed"
```

### External Secrets Operator

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "http://vault:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"
          serviceAccountRef:
            name: myapp
---
# External Secret for MongoDB credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mongodb-credentials
  namespace: production
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: vault-backend
    kind: SecretStore

  target:
    name: mongodb-secret
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        username: "{{ .username }}"
        password: "{{ .password }}"
        connection-string: "mongodb://{{ .username }}:{{ .password }}@mongodb-0.mongodb-headless:27017,mongodb-1.mongodb-headless:27017,mongodb-2.mongodb-headless:27017/myapp?replicaSet=rs0&authSource=admin"

  data:
    - secretKey: username
      remoteRef:
        key: mongodb/creds/app-role
        property: username

    - secretKey: password
      remoteRef:
        key: mongodb/creds/app-role
        property: password
---
# External Secret for static configuration
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mongodb-config-secret
  namespace: production
spec:
  refreshInterval: 24h

  secretStoreRef:
    name: vault-backend
    kind: SecretStore

  target:
    name: mongodb-config-values
    creationPolicy: Owner

  dataFrom:
    - extract:
        key: mongodb/config
    - extract:
        key: mongodb/connection
```

### Vault Agent Sidecar

```yaml
# deployment-vault-agent.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-vault
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"

        # Inject MongoDB credentials
        vault.hashicorp.com/agent-inject-secret-mongodb: "mongodb/creds/app-role"
        vault.hashicorp.com/agent-inject-template-mongodb: |
          {{- with secret "mongodb/creds/app-role" -}}
          export MONGODB_USERNAME="{{ .Data.username }}"
          export MONGODB_PASSWORD="{{ .Data.password }}"
          export MONGODB_URI="mongodb://{{ .Data.username }}:{{ .Data.password }}@mongodb-0.mongodb-headless:27017,mongodb-1.mongodb-headless:27017,mongodb-2.mongodb-headless:27017/myapp?replicaSet=rs0&authSource=admin"
          {{- end }}

        # Inject static configuration
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/mongodb/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/mongodb/config" -}}
          {
            "database": {
              "name": "{{ .Data.data.database_name }}",
              "maxPoolSize": {{ .Data.data.max_pool_size }},
              "minPoolSize": {{ .Data.data.min_pool_size }}
            },
            "cache": {
              "ttl": {{ .Data.data.cache_ttl }}
            }
          }
          {{- end }}

    spec:
      serviceAccountName: myapp

      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0

          command: ["/bin/sh", "-c"]
          args:
            - |
              # Source Vault secrets
              source /vault/secrets/mongodb

              # Start application
              exec node server.js

          volumeMounts:
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true

      volumes:
        - name: vault-secrets
          emptyDir:
            medium: Memory
```

---

## Sealed Secrets

### Installation et Configuration

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64
sudo install -m 755 kubeseal-linux-amd64 /usr/local/bin/kubeseal

# Get public key
kubeseal --fetch-cert > pub-cert.pem
```

### Creating Sealed Secrets

```bash
#!/bin/bash
# scripts/create-sealed-secret.sh

# Create a regular secret
kubectl create secret generic mongodb-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecurePassword123! \
  --dry-run=client \
  -o yaml > mongodb-secret.yaml

# Seal the secret
kubeseal --format yaml \
  --cert=pub-cert.pem \
  < mongodb-secret.yaml \
  > mongodb-sealed-secret.yaml

# Now mongodb-sealed-secret.yaml can be committed to git
```

### Sealed Secret Manifest

```yaml
# mongodb-sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mongodb-credentials
  namespace: production
spec:
  encryptedData:
    username: AgBHw8J7XETaa7vTHfXxHwUq8J...
    password: AgCKz3PLjQN8F5qKkN8NqVsRwL...
    connection-string: AgDxKmF4n8PjL9sKqN3VtYwRxM...
  template:
    metadata:
      name: mongodb-credentials
      namespace: production
    type: Opaque
```

---

## Configuration Dynamique

### Consul Integration

```go
// consul-config-loader.go
package config

import (
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/hashicorp/consul/api"
)

type MongoDBConfig struct {
    ConnectionString string `json:"connection_string"`
    Database         string `json:"database"`
    MaxPoolSize      int    `json:"max_pool_size"`
    MinPoolSize      int    `json:"min_pool_size"`
    CacheTTL         int    `json:"cache_ttl"`
}

type ConfigLoader struct {
    client      *api.Client
    kv          *api.KV
    configCache *MongoDBConfig
    stopChan    chan struct{}
}

func NewConfigLoader(consulAddr string) (*ConfigLoader, error) {
    config := api.DefaultConfig()
    config.Address = consulAddr

    client, err := api.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("failed to create Consul client: %w", err)
    }

    return &ConfigLoader{
        client:   client,
        kv:       client.KV(),
        stopChan: make(chan struct{}),
    }, nil
}

func (cl *ConfigLoader) LoadConfig(key string) (*MongoDBConfig, error) {
    pair, _, err := cl.kv.Get(key, nil)
    if err != nil {
        return nil, fmt.Errorf("failed to get config from Consul: %w", err)
    }

    if pair == nil {
        return nil, fmt.Errorf("config key not found: %s", key)
    }

    var config MongoDBConfig
    if err := json.Unmarshal(pair.Value, &config); err != nil {
        return nil, fmt.Errorf("failed to unmarshal config: %w", err)
    }

    cl.configCache = &config
    return &config, nil
}

func (cl *ConfigLoader) WatchConfig(key string, onChange func(*MongoDBConfig)) error {
    queryOpts := &api.QueryOptions{
        WaitIndex: 0,
        WaitTime:  time.Minute * 5,
    }

    go func() {
        for {
            select {
            case <-cl.stopChan:
                return
            default:
                pair, meta, err := cl.kv.Get(key, queryOpts)
                if err != nil {
                    log.Printf("Error watching config: %v", err)
                    time.Sleep(time.Second * 5)
                    continue
                }

                if pair == nil {
                    log.Printf("Config key not found: %s", key)
                    time.Sleep(time.Second * 5)
                    continue
                }

                // Update wait index for long polling
                queryOpts.WaitIndex = meta.LastIndex

                // Parse new config
                var newConfig MongoDBConfig
                if err := json.Unmarshal(pair.Value, &newConfig); err != nil {
                    log.Printf("Error unmarshaling config: %v", err)
                    continue
                }

                // Check if config changed
                if cl.configCache == nil || !cl.isConfigEqual(&newConfig, cl.configCache) {
                    log.Println("Config changed, updating...")
                    cl.configCache = &newConfig

                    // Notify callback
                    if onChange != nil {
                        onChange(&newConfig)
                    }
                }
            }
        }
    }()

    return nil
}

func (cl *ConfigLoader) isConfigEqual(a, b *MongoDBConfig) bool {
    return a.ConnectionString == b.ConnectionString &&
        a.Database == b.Database &&
        a.MaxPoolSize == b.MaxPoolSize &&
        a.MinPoolSize == b.MinPoolSize &&
        a.CacheTTL == b.CacheTTL
}

func (cl *ConfigLoader) Stop() {
    close(cl.stopChan)
}

// Usage example
func main() {
    loader, err := NewConfigLoader("localhost:8500")
    if err != nil {
        log.Fatal(err)
    }
    defer loader.Stop()

    // Load initial config
    config, err := loader.LoadConfig("mongodb/config")
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Initial config: %+v", config)

    // Watch for changes
    err = loader.WatchConfig("mongodb/config", func(newConfig *MongoDBConfig) {
        log.Printf("Config updated: %+v", newConfig)

        // Reconnect to MongoDB with new config
        // updateMongoConnection(newConfig)
    })

    if err != nil {
        log.Fatal(err)
    }

    // Keep running
    select {}
}
```

### etcd Configuration

```go
// etcd-config-manager.go
package config

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    clientv3 "go.etcd.io/etcd/client/v3"
)

type EtcdConfigManager struct {
    client *clientv3.Client
    kv     clientv3.KV
}

func NewEtcdConfigManager(endpoints []string) (*EtcdConfigManager, error) {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout: 5 * time.Second,
    })

    if err != nil {
        return nil, fmt.Errorf("failed to create etcd client: %w", err)
    }

    return &EtcdConfigManager{
        client: client,
        kv:     clientv3.NewKV(client),
    }, nil
}

func (m *EtcdConfigManager) PutConfig(ctx context.Context, key string, config interface{}) error {
    data, err := json.Marshal(config)
    if err != nil {
        return fmt.Errorf("failed to marshal config: %w", err)
    }

    _, err = m.kv.Put(ctx, key, string(data))
    if err != nil {
        return fmt.Errorf("failed to put config: %w", err)
    }

    return nil
}

func (m *EtcdConfigManager) GetConfig(ctx context.Context, key string, config interface{}) error {
    resp, err := m.kv.Get(ctx, key)
    if err != nil {
        return fmt.Errorf("failed to get config: %w", err)
    }

    if len(resp.Kvs) == 0 {
        return fmt.Errorf("config not found: %s", key)
    }

    err = json.Unmarshal(resp.Kvs[0].Value, config)
    if err != nil {
        return fmt.Errorf("failed to unmarshal config: %w", err)
    }

    return nil
}

func (m *EtcdConfigManager) WatchConfig(ctx context.Context, key string, onChange func(interface{})) error {
    watchChan := m.client.Watch(ctx, key)

    go func() {
        for watchResp := range watchChan {
            for _, event := range watchResp.Events {
                var config map[string]interface{}
                if err := json.Unmarshal(event.Kv.Value, &config); err != nil {
                    log.Printf("Error unmarshaling config: %v", err)
                    continue
                }

                log.Printf("Config changed: %s", event.Type)
                if onChange != nil {
                    onChange(config)
                }
            }
        }
    }()

    return nil
}

func (m *EtcdConfigManager) Close() error {
    return m.client.Close()
}
```

---

## Configuration Validation

### JSON Schema Validation

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "MongoDB Configuration Schema",
  "type": "object",
  "required": ["database", "connection"],
  "properties": {
    "database": {
      "type": "object",
      "required": ["name", "maxPoolSize", "minPoolSize"],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1,
          "maxLength": 64,
          "pattern": "^[a-zA-Z0-9_-]+$"
        },
        "maxPoolSize": {
          "type": "integer",
          "minimum": 1,
          "maximum": 1000
        },
        "minPoolSize": {
          "type": "integer",
          "minimum": 0,
          "maximum": 100
        },
        "connectionTimeout": {
          "type": "integer",
          "minimum": 1000,
          "maximum": 300000,
          "default": 30000
        },
        "socketTimeout": {
          "type": "integer",
          "minimum": 1000,
          "maximum": 300000,
          "default": 30000
        }
      }
    },
    "connection": {
      "type": "object",
      "required": ["uri"],
      "properties": {
        "uri": {
          "type": "string",
          "pattern": "^mongodb(\\+srv)?:\\/\\/.+"
        },
        "replicaSet": {
          "type": "string",
          "minLength": 1
        },
        "authSource": {
          "type": "string",
          "enum": ["admin", "local"]
        },
        "retryWrites": {
          "type": "boolean",
          "default": true
        },
        "w": {
          "oneOf": [
            {"type": "integer", "minimum": 0},
            {"type": "string", "enum": ["majority"]}
          ],
          "default": "majority"
        }
      }
    },
    "cache": {
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean",
          "default": true
        },
        "ttl": {
          "type": "integer",
          "minimum": 0,
          "maximum": 3600,
          "default": 300
        },
        "maxSize": {
          "type": "integer",
          "minimum": 0,
          "maximum": 100000,
          "default": 1000
        }
      }
    },
    "logging": {
      "type": "object",
      "properties": {
        "level": {
          "type": "string",
          "enum": ["debug", "info", "warn", "error"],
          "default": "info"
        },
        "format": {
          "type": "string",
          "enum": ["json", "text"],
          "default": "json"
        }
      }
    }
  }
}
```

### Validation Script

```javascript
// config-validator.js
const Ajv = require('ajv');
const addFormats = require('ajv-formats');
const fs = require('fs');

class ConfigValidator {
  constructor(schemaPath) {
    this.ajv = new Ajv({ allErrors: true });
    addFormats(this.ajv);

    const schema = JSON.parse(fs.readFileSync(schemaPath, 'utf8'));
    this.validate = this.ajv.compile(schema);
  }

  validateConfig(config) {
    const valid = this.validate(config);

    if (!valid) {
      const errors = this.validate.errors.map(err => ({
        path: err.instancePath,
        message: err.message,
        params: err.params
      }));

      return {
        valid: false,
        errors
      };
    }

    return { valid: true };
  }

  validateFile(configPath) {
    const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
    return this.validateConfig(config);
  }
}

// CLI usage
if (require.main === module) {
  const validator = new ConfigValidator('./schema.json');
  const result = validator.validateFile(process.argv[2]);

  if (result.valid) {
    console.log('✓ Configuration is valid');
    process.exit(0);
  } else {
    console.error('✗ Configuration is invalid:');
    result.errors.forEach(err => {
      console.error(`  - ${err.path}: ${err.message}`);
    });
    process.exit(1);
  }
}

module.exports = ConfigValidator;
```

---

## Outils de Gestion

### Kustomize Overlays

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

configMapGenerator:
  - name: app-config
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=info
---
# overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: development

# Patch configuration for dev
configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - NODE_ENV=development
      - LOG_LEVEL=debug
      - ENABLE_PROFILING=true

# Patch deployment
patchesStrategicMerge:
  - deployment-patch.yaml

replicas:
  - name: myapp
    count: 1
---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: production

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=warn
      - CACHE_ENABLED=true

replicas:
  - name: myapp
    count: 5

# Add resources specific to production
resources:
  - hpa.yaml
  - pdb.yaml
```

### Helm Values Hierarchy

```yaml
# values.yaml (default)
replicaCount: 3

image:
  repository: myregistry/myapp
  tag: latest
  pullPolicy: IfNotPresent

mongodb:
  enabled: true
  uri: "mongodb://mongodb:27017"
  database: "myapp"

config:
  logLevel: info
  cacheEnabled: true
  cacheTTL: 300

resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
---
# values-dev.yaml
replicaCount: 1

mongodb:
  uri: "mongodb://mongodb-dev:27017"
  database: "myapp_dev"

config:
  logLevel: debug
  cacheEnabled: false
  enableProfiling: true

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
---
# values-prod.yaml
replicaCount: 5

mongodb:
  uri: "mongodb://mongodb-0.mongodb-headless:27017,mongodb-1.mongodb-headless:27017,mongodb-2.mongodb-headless:27017"
  database: "myapp"
  replicaSet: "rs0"

config:
  logLevel: warn
  cacheEnabled: true
  cacheTTL: 600

resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "2Gi"
    cpu: "2000m"

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
```

---

## Best Practices

### Checklist Configuration Management

```yaml
# configuration-best-practices.yaml
---
security:
  - ✅ Secrets jamais commitées en clair dans Git
  - ✅ Rotation automatique des secrets
  - ✅ Encryption at rest (Vault, Sealed Secrets)
  - ✅ RBAC sur accès aux secrets
  - ✅ Audit logging activé
  - ✅ Principe du moindre privilège
  - ✅ TLS pour communication avec config stores

separation:
  - ✅ Configuration séparée du code
  - ✅ Environnements isolés (dev/staging/prod)
  - ✅ ConfigMaps pour config non-sensible
  - ✅ Secrets pour données sensibles
  - ✅ External Secrets pour centralisation

validation:
  - ✅ Schema validation JSON/YAML
  - ✅ Tests automatiques de configuration
  - ✅ Dry-run avant déploiement
  - ✅ Validation dans CI/CD pipeline
  - ✅ Type checking (TypeScript, Go structs)

versioning:
  - ✅ Configuration versionnée avec Git
  - ✅ Tags pour releases
  - ✅ Rollback capability
  - ✅ Change tracking
  - ✅ Documentation des changements

dynamic_config:
  - ✅ Hot-reload pour configs non-critiques
  - ✅ Graceful degradation si config unavailable
  - ✅ Default values sains
  - ✅ Circuit breakers
  - ✅ Health checks intégrés

monitoring:
  - ✅ Alertes sur changements de config
  - ✅ Metrics sur config loading
  - ✅ Logs des modifications
  - ✅ Dashboard de config actuelle
  - ✅ Audit trail complet

testing:
  - ✅ Tests unitaires sur config loading
  - ✅ Integration tests par environnement
  - ✅ Smoke tests post-déploiement
  - ✅ Config validation automatisée
  - ✅ Rollback testé régulièrement

documentation:
  - ✅ README par environnement
  - ✅ Variables documentées
  - ✅ Exemples de configuration
  - ✅ Troubleshooting guide
  - ✅ Contact d'escalation
```

---

## Conclusion

La gestion des configurations est critique pour des applications MongoDB fiables et sécurisées :

**Outils clés :**
- **Kubernetes** : ConfigMaps et Secrets natifs
- **Vault** : Gestion centralisée des secrets avec rotation
- **Sealed Secrets** : Secrets encryptés dans Git
- **External Secrets** : Synchronisation automatique
- **Consul/etcd** : Configuration dynamique

**Principes essentiels :**
- Séparation code/config
- Secrets jamais en clair
- Validation automatique
- Environnements isolés
- Configuration immuable

**Pour MongoDB :**
- Connection strings dans Secrets
- Configuration performance dans ConfigMaps
- Certificats TLS gérés par Vault/cert-manager
- Rotation automatique des credentials
- Monitoring des changements

**Stratégies avancées :**
- Dynamic configuration avec watch
- Feature flags pour transitions
- Multi-environment avec overlays
- GitOps workflows
- Automated validation

Une approche disciplinée de la gestion des configurations garantit la sécurité, la fiabilité et la maintenabilité des déploiements MongoDB en production.

---


⏭️ [Environnements multi-régions](/18-devops-deploiement/11-environnements-multi-regions.md)
