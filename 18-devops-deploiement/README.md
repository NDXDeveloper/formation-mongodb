🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 18 : DevOps et Déploiement

## Vue d'ensemble

Le déploiement et l'opération de MongoDB en environnement de production nécessitent une approche DevOps moderne, combinant automatisation, reproductibilité et observabilité. Ce chapitre explore les pratiques, outils et architectures permettant de déployer, gérer et maintenir MongoDB à l'échelle dans des environnements cloud-native et hybrides.

L'objectif est de maîtriser l'orchestration de clusters MongoDB complexes (Replica Sets, Sharding) à travers différentes plateformes (conteneurs, Kubernetes, cloud providers), tout en garantissant la sécurité, la performance et la résilience des déploiements.

---

## Contexte et Enjeux

### Défis du Déploiement MongoDB en Production

Le déploiement de MongoDB en production présente des défis spécifiques qui nécessitent une approche DevOps mature :

**1. Gestion de l'état distribué**
- MongoDB est une base de données **stateful** avec des contraintes de persistance
- La réplication nécessite une orchestration précise des membres du Replica Set
- Le sharding introduit une complexité topologique (shards, config servers, mongos)
- Les données ne peuvent pas être simplement recréées comme pour des applications stateless

**2. Haute disponibilité et résilience**
- Gestion automatique du failover et de l'élection du Primary
- Maintien du quorum lors des opérations de maintenance
- Stratégies de placement multi-zones et multi-régions
- Recovery automatique en cas de défaillance

**3. Scalabilité et performance**
- Provisionnement dynamique des ressources (CPU, RAM, IOPS)
- Gestion du balancing des chunks dans les clusters shardés
- Optimisation du placement des workloads (read preference, write concern)
- Monitoring proactif et auto-scaling

**4. Cohérence opérationnelle**
- Reproductibilité des déploiements (dev, staging, production)
- Versioning de l'infrastructure et des configurations
- Gestion des secrets et des credentials
- Traçabilité des changements (audit trail)

---

## Principes Fondamentaux du DevOps MongoDB

### Infrastructure as Code (IaC)

L'Infrastructure as Code est le pilier du DevOps moderne pour MongoDB :

```
┌─────────────────────────────────────────────────────────────┐
│                    Infrastructure as Code                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Terraform   │  │   Ansible    │  │ Kubernetes   │       │
│  │              │  │              │  │   Operators  │       │
│  │  • Provision │  │  • Configure │  │              │       │
│  │  • Network   │  │  • Deploy    │  │  • Orchestrate       │
│  │  • Storage   │  │  • Maintain  │  │  • Scale             │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                  │                  │             │
│         └──────────────────┴──────────────────┘             │
│                            ▼                                │
│              ┌──────────────────────────┐                   │
│              │   MongoDB Cluster        │                   │
│              │   (Replica Set/Sharded)  │                   │
│              └──────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

**Avantages de l'IaC pour MongoDB :**
- **Reproductibilité** : Déploiement identique sur tous les environnements
- **Versioning** : Historique complet des changements d'infrastructure
- **Testing** : Validation des configurations avant application
- **Documentation** : Le code est la documentation
- **Collaboration** : Review et validation collaborative (Git workflow)

### Architecture de Référence DevOps

```
┌──────────────────────────────────────────────────────────────────┐
│                         CI/CD Pipeline                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Git Repository                                                  │
│  ├── infrastructure/                                             │
│  │   ├── terraform/          (Provisioning)                      │
│  │   ├── ansible/            (Configuration)                     │
│  │   └── kubernetes/         (Orchestration)                     │
│  ├── monitoring/                                                 │
│  │   ├── prometheus/                                             │
│  │   └── grafana/                                                │
│  └── backup/                                                     │
│      └── scripts/                                                │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │   Validate   │───▶│     Test     │───▶│    Deploy    │        │
│  │              │    │              │    │              │        │
│  │  • Linting   │    │  • Dry-run   │    │  • Staging   │        │
│  │  • Security  │    │  • Integration    │  • Production│        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Production Environment                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Kubernetes Cluster                            │  │
│  │                                                            │  │
│  │  ┌──────────────────┐         ┌──────────────────┐         │  │
│  │  │  MongoDB Operator          │   Monitoring     │         │  │
│  │  │                  │         │   Stack          │         │  │
│  │  │  • StatefulSets  │         │                  │         │  │
│  │  │  • Services      │◀────────│  • Prometheus    │         │  │
│  │  │  • PVCs          │         │  • Grafana       │         │  │
│  │  │  • ConfigMaps    │         │  • Alertmanager  │         │  │
│  │  └──────────────────┘         └──────────────────┘         │  │
│  │           │                              │                 │  │
│  │           ▼                              ▼                 │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │         MongoDB Replica Set / Sharded Cluster       │   │  │
│  │  │                                                     │   │  │
│  │  │  Primary ──┬── Secondary ──┬── Secondary ──┬── ...  │   │  │
│  │  │            │               │               │        │   │  │
│  │  │         [PVC]            [PVC]            [PVC]     │   │  │
│  │  └─────────────────────────────────────────────────────┘   │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Stratégies de Déploiement

### 1. Déploiement Conteneurisé (Docker)

**Cas d'usage :**
- Environnements de développement et de test
- Déploiements simples sur VM ou bare metal
- Prototypage rapide
- CI/CD pipelines

**Architecture typique :**

```yaml
# Exemple de stack Docker Compose pour Replica Set
version: '3.8'

services:
  mongo-primary:
    image: mongo:7.0
    container_name: mongo-primary
    command: mongod --replSet rs0 --bind_ip_all
    volumes:
      - mongo-primary-data:/data/db
    networks:
      - mongo-cluster
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}

  mongo-secondary-1:
    image: mongo:7.0
    container_name: mongo-secondary-1
    command: mongod --replSet rs0 --bind_ip_all
    volumes:
      - mongo-secondary-1-data:/data/db
    networks:
      - mongo-cluster
    depends_on:
      - mongo-primary

  mongo-secondary-2:
    image: mongo:7.0
    container_name: mongo-secondary-2
    command: mongod --replSet rs0 --bind_ip_all
    volumes:
      - mongo-secondary-2-data:/data/db
    networks:
      - mongo-cluster
    depends_on:
      - mongo-primary

volumes:
  mongo-primary-data:
  mongo-secondary-1-data:
  mongo-secondary-2-data:

networks:
  mongo-cluster:
    driver: bridge
```

**Limitations :**
- Pas d'orchestration automatique du failover
- Gestion manuelle du scaling
- Pas de self-healing
- Adapté aux environnements non-critiques

### 2. Déploiement Kubernetes

**Cas d'usage :**
- Production à l'échelle
- Multi-tenant
- Cloud-native applications
- Auto-scaling et self-healing requis

**Composants clés :**

```
┌───────────────────────────────────────────────────────────┐
│              Kubernetes MongoDB Architecture              │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │         MongoDB Kubernetes Operator                 │  │
│  │                                                     │  │
│  │  • CRD (Custom Resource Definitions)                │  │
│  │  • Controller (reconciliation loop)                 │  │
│  │  • Webhook (admission control)                      │  │
│  └─────────────────────────────────────────────────────┘  │
│                          │                                │
│                          ▼                                │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              MongoDB Custom Resource                │  │
│  │                                                     │  │
│  │  apiVersion: mongodbcommunity.mongodb.com/v1        │  │
│  │  kind: MongoDBCommunity                             │  │
│  │  metadata:                                          │  │
│  │    name: production-replica-set                     │  │
│  │  spec:                                              │  │
│  │    members: 3                                       │  │
│  │    type: ReplicaSet                                 │  │
│  │    version: "7.0.0"                                 │  │
│  │    security:                                        │  │
│  │      authentication:                                │  │
│  │        modes: ["SCRAM"]                             │  │
│  └─────────────────────────────────────────────────────┘  │
│                          │                                │
│                          ▼                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ StatefulSet  │  │   Service    │  │     PVC      │     │
│  │              │  │              │  │              │     │
│  │  • Pods      │  │  • Headless  │  │  • Storage   │     │
│  │  • Ordering  │  │  • Discovery │  │  • Snapshots │     │
│  │  • Identity  │  │  • Load Bal. │  │  • Backup    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Avantages Kubernetes :**
- **Self-healing** : Redémarrage automatique des pods défaillants
- **Auto-scaling** : HPA (Horizontal Pod Autoscaler) et VPA (Vertical Pod Autoscaler)
- **Rolling updates** : Mise à jour sans interruption de service
- **Multi-zone** : Distribution des pods sur plusieurs zones de disponibilité
- **Declarative** : Infrastructure déclarée via YAML
- **Ecosystem** : Intégration avec monitoring, logging, service mesh

### 3. Déploiement Cloud-Native

**MongoDB Atlas vs Self-Managed :**

| Critère | MongoDB Atlas | Self-Managed K8s | Self-Managed VM |
|---------|---------------|------------------|-----------------|
| **Gestion** | Fully managed | Semi-managed | Fully self-managed |
| **Scaling** | Automatique | Via Operator | Manuel |
| **Backup** | Intégré (PITR) | À configurer | À configurer |
| **Monitoring** | Intégré | Prometheus/Grafana | À configurer |
| **Coût** | Pay-as-you-go | Infrastructure + Ops | Infrastructure + Ops |
| **Flexibilité** | Limitée | Élevée | Totale |
| **Time to market** | Immédiat | Rapide | Lent |
| **Expertise requise** | Faible | Moyenne | Élevée |

**Approche hybride recommandée :**

```
┌─────────────────────────────────────────────────────────────┐
│                    Hybrid Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐        ┌─────────────────────┐     │
│  │   MongoDB Atlas     │        │   Self-Managed K8s  │     │
│  │   (Production)      │◀──────▶│   (Dev/Staging)     │     │
│  │                     │        │                     │     │
│  │  • Multi-region     │        │  • On-premise       │     │
│  │  • Auto-backup      │        │  • Full control     │     │
│  │  • Managed upgrades │        │  • Cost-effective   │     │
│  └─────────────────────┘        └─────────────────────┘     │
│           │                              │                  │
│           └──────────────┬───────────────┘                  │
│                          ▼                                  │
│              ┌───────────────────────┐                      │
│              │   Data Sync / CDC     │                      │
│              │   (Change Streams)    │                      │
│              └───────────────────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Patterns d'Orchestration

### Pattern 1 : Replica Set avec StatefulSet

**Objectif :** Déployer un Replica Set MongoDB résilient dans Kubernetes

**Architecture :**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-replica-set
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: mongodb
        image: mongo:7.0
        command:
          - mongod
          - "--replSet"
          - rs0
          - "--bind_ip_all"
        ports:
        - containerPort: 27017
          name: mongodb
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        livenessProbe:
          exec:
            command:
              - mongosh
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
              - mongosh
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi
```

**Service Headless pour Service Discovery :**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    app: mongodb
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None  # Headless service
  selector:
    app: mongodb
```

**Caractéristiques clés :**
- **Identité stable** : Chaque pod a un hostname DNS prévisible (`mongodb-replica-set-0.mongodb-service`)
- **Ordering** : Démarrage et arrêt séquentiel des pods
- **Persistent Storage** : Un PVC par pod, conservé lors des redémarrages
- **Service Discovery** : DNS interne pour la communication inter-pods

### Pattern 2 : Sharded Cluster avec Operator

**Objectif :** Déployer un cluster shardé avec MongoDB Kubernetes Operator

**Architecture globale :**

```
┌───────────────────────────────────────────────────────────────┐
│                  Sharded Cluster on Kubernetes                │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Mongos (Routers)                      │ │
│  │                                                          │ │
│  │   ┌─────────┐   ┌─────────┐   ┌─────────┐                │ │
│  │   │ mongos-0│   │ mongos-1│   │ mongos-2│                │ │
│  │   └────┬────┘   └────┬────┘   └────┬────┘                │ │
│  │        │             │             │                     │ │
│  └────────┼─────────────┼─────────────┼─────────────────────┘ │
│           │             │             │                       │
│           └─────────────┼─────────────┘                       │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              Config Server Replica Set                   │ │
│  │                                                          │ │
│  │   ┌──────────┐   ┌──────────┐   ┌──────────┐             │ │
│  │   │ config-0 │───│ config-1 │───│ config-2 │             │ │
│  │   └──────────┘   └──────────┘   └──────────┘             │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         │                                     │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Shard Replica Sets                    │ │
│  │                                                          │ │
│  │  ┌────────────────────┐     ┌────────────────────┐       │ │
│  │  │   Shard 0 RS       │     │   Shard 1 RS       │       │ │
│  │  │  ┌────┐ ┌────┐     │     │  ┌────┐ ┌────┐     │       │ │
│  │  │  │ P  │ │ S  │ ... │     │  │ P  │ │ S  │ ... │       │ │
│  │  │  └────┘ └────┘     │     │  └────┘ └────┘     │       │ │
│  │  └────────────────────┘     └────────────────────┘       │ │
│  │                                                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Custom Resource Definition :**

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: sharded-cluster
spec:
  type: ShardedCluster
  version: "7.0.0"

  shardCount: 2

  configServer:
    members: 3
    resources:
      requests:
        memory: 2Gi
        cpu: 1
      limits:
        memory: 4Gi
        cpu: 2
    storage:
      size: 50Gi
      storageClass: fast-ssd

  mongos:
    count: 3
    resources:
      requests:
        memory: 1Gi
        cpu: 0.5
      limits:
        memory: 2Gi
        cpu: 1

  shards:
    - name: shard0
      members: 3
      resources:
        requests:
          memory: 8Gi
          cpu: 4
        limits:
          memory: 16Gi
          cpu: 8
      storage:
        size: 500Gi
        storageClass: fast-ssd

    - name: shard1
      members: 3
      resources:
        requests:
          memory: 8Gi
          cpu: 4
        limits:
          memory: 16Gi
          cpu: 8
      storage:
        size: 500Gi
        storageClass: fast-ssd

  security:
    authentication:
      modes: ["SCRAM"]
    tls:
      enabled: true
      certificateKeySecretRef:
        name: mongodb-tls-secret
```

### Pattern 3 : Multi-Region Deployment

**Objectif :** Déployer MongoDB dans plusieurs régions pour la haute disponibilité et la réduction de la latence

**Architecture :**

```
┌────────────────────────────────────────────────────────────────────┐
│                    Global Multi-Region Architecture                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Region EU-West │  │   Region US-East │  │   Region AP-South│  │
│  │   (Paris)        │  │   (Virginia)     │  │   (Singapore)    │  │
│  │                  │  │                  │  │                  │  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │
│  │  │ K8s Node   │  │  │  │ K8s Node   │  │  │  │ K8s Node   │  │  │
│  │  │            │  │  │  │            │  │  │  │            │  │  │
│  │  │ ┌────────┐ │  │  │  │ ┌────────┐ │  │  │  │ ┌────────┐ │  │  │
│  │  │ │Primary │ │  │  │  │ │Secondary │  │  │  │ │Secondary │  │  │
│  │  │ │(P)     │◀┼──┼──┼──┼▶│(S)     │◀┼──┼──┼──┼▶│(S)     │ │  │  │
│  │  │ └────────┘ │  │  │  │ └────────┘ │  │  │  │ └────────┘ │  │  │
│  │  │            │  │  │  │            │  │  │  │            │  │  │
│  │  │ Priority=5 │  │  │  │ Priority=3 │  │  │  │ Priority=1 │  │  │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │  │
│  │                  │  │                  │  │                  │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│           │                     │                     │            │
│           └─────────────────────┴─────────────────────┘            │
│                         Low-latency network                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

**Configuration Terraform pour multi-région :**

```hcl
# terraform/multi-region/main.tf

variable "regions" {
  type = map(object({
    cluster_name = string
    priority     = number
    node_count   = number
  }))
  default = {
    "eu-west-3" = {
      cluster_name = "paris-cluster"
      priority     = 5  # Highest - Primary preferred
      node_count   = 3
    }
    "us-east-1" = {
      cluster_name = "virginia-cluster"
      priority     = 3
      node_count   = 3
    }
    "ap-south-1" = {
      cluster_name = "singapore-cluster"
      priority     = 1
      node_count   = 2
    }
  }
}

# Deploy Kubernetes clusters in each region
module "kubernetes_clusters" {
  for_each = var.regions
  source   = "./modules/eks-cluster"

  region       = each.key
  cluster_name = each.value.cluster_name
  node_count   = each.value.node_count
}

# Deploy MongoDB Operator in each cluster
module "mongodb_operator" {
  for_each   = module.kubernetes_clusters
  source     = "./modules/mongodb-operator"

  kubeconfig = each.value.kubeconfig
}

# Configure Multi-Region Replica Set
resource "kubernetes_manifest" "mongodb_multi_region" {
  provider = kubernetes.paris

  manifest = {
    apiVersion = "mongodbcommunity.mongodb.com/v1"
    kind       = "MongoDBCommunity"
    metadata = {
      name      = "global-replica-set"
      namespace = "mongodb"
    }
    spec = {
      members = 8
      type    = "ReplicaSet"
      version = "7.0.0"

      memberConfig = [
        {
          host     = "mongodb-0.paris.mongodb.svc.cluster.local"
          priority = 5
          tags     = { region = "eu-west" }
        },
        {
          host     = "mongodb-1.virginia.mongodb.svc.cluster.local"
          priority = 3
          tags     = { region = "us-east" }
        },
        {
          host     = "mongodb-2.singapore.mongodb.svc.cluster.local"
          priority = 1
          tags     = { region = "ap-south" }
        }
      ]

      security = {
        authentication = {
          modes = ["SCRAM"]
        }
        tls = {
          enabled = true
        }
      }
    }
  }
}
```

**Read Preference pour la géolocalisation :**

```javascript
// Application configuration
const client = new MongoClient(uri, {
  readPreference: {
    mode: 'nearest',
    tags: [
      { region: process.env.AWS_REGION },  // Prefer same region
      { region: 'eu-west' },                // Fallback to EU
      {}                                     // Any member as last resort
    ],
    maxStalenessSeconds: 90
  }
});
```

---

## Stratégies de Configuration

### Configuration Management avec Ansible

**Structure de projet Ansible :**

```
ansible/
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── mongodb_primary.yml
│   │       └── mongodb_secondary.yml
│   └── staging/
│       └── hosts.yml
├── roles/
│   ├── mongodb_install/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   └── configure.yml
│   │   ├── templates/
│   │   │   ├── mongod.conf.j2
│   │   │   └── systemd.service.j2
│   │   └── defaults/
│   │       └── main.yml
│   ├── mongodb_replica_set/
│   │   └── tasks/
│   │       ├── main.yml
│   │       ├── initiate.yml
│   │       └── add_members.yml
│   └── mongodb_monitoring/
│       └── tasks/
│           └── main.yml
└── playbooks/
    ├── deploy_replica_set.yml
    ├── upgrade_mongodb.yml
    └── backup_configure.yml
```

**Playbook de déploiement Replica Set :**

```yaml
# playbooks/deploy_replica_set.yml
---
- name: Deploy MongoDB Replica Set
  hosts: mongodb_replica_set
  become: yes
  vars:
    mongodb_version: "7.0"
    replica_set_name: "production-rs"
    mongodb_port: 27017

  pre_tasks:
    - name: Validate inventory
      assert:
        that:
          - groups['mongodb_primary'] | length == 1
          - groups['mongodb_secondary'] | length >= 2
        fail_msg: "Replica Set requires 1 primary and at least 2 secondaries"

  roles:
    - role: mongodb_install
      tags: ['install']

    - role: mongodb_security
      tags: ['security']

    - role: mongodb_replica_set
      tags: ['replication']
      when: inventory_hostname in groups['mongodb_primary']

  post_tasks:
    - name: Wait for Replica Set to stabilize
      command: >
        mongosh --eval "rs.status().ok"
      register: rs_status
      until: rs_status.rc == 0
      retries: 10
      delay: 5
      when: inventory_hostname in groups['mongodb_primary']
```

**Template de configuration mongod.conf :**

```jinja2
# templates/mongod.conf.j2
# MongoDB Configuration File - Generated by Ansible

# Storage
storage:
  dbPath: {{ mongodb_dbpath }}
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: {{ mongodb_cache_size_gb }}
      journalCompressor: {{ mongodb_journal_compressor }}
    collectionConfig:
      blockCompressor: {{ mongodb_block_compressor }}
    indexConfig:
      prefixCompression: true

# Replication
replication:
  replSetName: {{ replica_set_name }}
  oplogSizeMB: {{ mongodb_oplog_size_mb }}

# Network
net:
  port: {{ mongodb_port }}
  bindIp: {{ mongodb_bind_ip }}
  maxIncomingConnections: {{ mongodb_max_connections }}
  tls:
    mode: requireTLS
    certificateKeyFile: {{ mongodb_tls_cert_path }}
    CAFile: {{ mongodb_tls_ca_path }}

# Security
security:
  authorization: enabled
  keyFile: {{ mongodb_keyfile_path }}
  javascriptEnabled: false

# Operation Profiling
operationProfiling:
  mode: {{ mongodb_profiling_mode }}
  slowOpThresholdMs: {{ mongodb_slow_op_threshold }}

# System Log
systemLog:
  destination: file
  path: {{ mongodb_log_path }}
  logAppend: true
  logRotate: reopen
  verbosity: {{ mongodb_log_verbosity }}
  component:
    replication:
      verbosity: {{ mongodb_replication_verbosity }}

# Process Management
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: false  # systemd manages the process

# Resource Control
setParameter:
  diagnosticDataCollectionEnabled: {{ mongodb_diagnostic_data_enabled }}
  enableLocalhostAuthBypass: false
  maxIndexBuildMemoryUsageMegabytes: {{ mongodb_index_build_memory_mb }}
```

### Infrastructure as Code avec Terraform

**Module Terraform pour MongoDB Atlas :**

```hcl
# terraform/modules/mongodb-atlas/main.tf

terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.12"
    }
  }
}

variable "project_name" {
  description = "MongoDB Atlas Project Name"
  type        = string
}

variable "cluster_config" {
  description = "Cluster configuration"
  type = object({
    name                     = string
    tier                     = string
    region                   = string
    mongodb_major_version    = string
    backup_enabled           = bool
    auto_scaling_enabled     = bool
    pit_enabled              = bool
  })
}

# MongoDB Atlas Project
resource "mongodbatlas_project" "main" {
  name   = var.project_name
  org_id = var.atlas_org_id
}

# Network Peering
resource "mongodbatlas_network_peering" "aws" {
  project_id             = mongodbatlas_project.main.id
  container_id           = mongodbatlas_network_container.main.id
  provider_name          = "AWS"
  vpc_id                 = var.aws_vpc_id
  aws_account_id         = var.aws_account_id
  route_table_cidr_block = var.vpc_cidr_block
}

# Cluster
resource "mongodbatlas_cluster" "main" {
  project_id = mongodbatlas_project.main.id
  name       = var.cluster_config.name

  # Provider settings
  provider_name               = "AWS"
  provider_region_name        = var.cluster_config.region
  provider_instance_size_name = var.cluster_config.tier

  # MongoDB version
  mongo_db_major_version = var.cluster_config.mongodb_major_version

  # Cluster configuration
  cluster_type = "REPLICASET"

  replication_specs {
    num_shards = 1

    regions_config {
      region_name     = var.cluster_config.region
      priority        = 7
      electable_nodes = 3
      read_only_nodes = 0
      analytics_nodes = 0
    }
  }

  # Auto-scaling
  auto_scaling_disk_gb_enabled            = var.cluster_config.auto_scaling_enabled
  auto_scaling_compute_enabled            = var.cluster_config.auto_scaling_enabled
  auto_scaling_compute_scale_down_enabled = var.cluster_config.auto_scaling_enabled

  # Backup
  backup_enabled                  = var.cluster_config.backup_enabled
  pit_enabled                     = var.cluster_config.pit_enabled
  cloud_backup                    = var.cluster_config.backup_enabled

  # Advanced configuration
  advanced_configuration {
    javascript_enabled                   = false
    minimum_enabled_tls_protocol         = "TLS1_2"
    no_table_scan                        = false
    oplog_size_mb                        = 2048
    sample_size_bi_connector            = 5000
    sample_refresh_interval_bi_connector = 300
  }

  # Encryption at rest
  encryption_at_rest_provider = "AWS"

  depends_on = [
    mongodbatlas_network_peering.aws
  ]
}

# Database Users
resource "mongodbatlas_database_user" "app_user" {
  username           = var.app_username
  password           = var.app_password
  project_id         = mongodbatlas_project.main.id
  auth_database_name = "admin"

  roles {
    role_name     = "readWrite"
    database_name = var.app_database_name
  }

  scopes {
    name = mongodbatlas_cluster.main.name
    type = "CLUSTER"
  }
}

# IP Access List
resource "mongodbatlas_project_ip_access_list" "k8s_nodes" {
  for_each = toset(var.kubernetes_node_ips)

  project_id = mongodbatlas_project.main.id
  ip_address = each.value
  comment    = "Kubernetes node ${each.value}"
}

# Monitoring and Alerts
resource "mongodbatlas_alert_configuration" "cpu_usage" {
  project_id = mongodbatlas_project.main.id
  enabled    = true

  event_type = "OUTSIDE_METRIC_THRESHOLD"

  threshold {
    operator  = "GREATER_THAN"
    threshold = 90
    units     = "RAW"
  }

  metric_threshold {
    metric_name = "SYSTEM_CPU_USER"
    operator    = "GREATER_THAN"
    threshold   = 90
    units       = "RAW"
    mode        = "AVERAGE"
  }

  notification {
    type_name     = "SLACK"
    interval_min  = 5
    delay_min     = 0
    slack_api_token   = var.slack_api_token
    slack_channel_name = var.slack_channel
  }
}

# Outputs
output "connection_string" {
  value     = mongodbatlas_cluster.main.connection_strings[0].standard_srv
  sensitive = true
}

output "cluster_id" {
  value = mongodbatlas_cluster.main.cluster_id
}
```

---

## Pipeline CI/CD

### GitLab CI/CD pour MongoDB

```yaml
# .gitlab-ci.yml

stages:
  - validate
  - test
  - deploy-staging
  - deploy-production

variables:
  TERRAFORM_VERSION: "1.6.0"
  KUBECTL_VERSION: "1.28.0"

# Validation stage
terraform:validate:
  stage: validate
  image: hashicorp/terraform:$TERRAFORM_VERSION
  script:
    - cd terraform/
    - terraform init -backend=false
    - terraform fmt -check
    - terraform validate
  only:
    changes:
      - terraform/**/*

ansible:lint:
  stage: validate
  image: willhallonline/ansible:latest
  script:
    - cd ansible/
    - ansible-lint playbooks/
    - yamllint .
  only:
    changes:
      - ansible/**/*

kubernetes:validate:
  stage: validate
  image: bitnami/kubectl:$KUBECTL_VERSION
  script:
    - kubectl apply --dry-run=client -f kubernetes/
    - kubectl kustomize kubernetes/ | kubectl apply --dry-run=server -f -
  only:
    changes:
      - kubernetes/**/*

# Test stage
terraform:plan:
  stage: test
  image: hashicorp/terraform:$TERRAFORM_VERSION
  script:
    - cd terraform/environments/staging/
    - terraform init
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - terraform/environments/staging/plan.tfplan
    expire_in: 1 day
  only:
    - merge_requests

# Deploy to staging
deploy:staging:
  stage: deploy-staging
  image: hashicorp/terraform:$TERRAFORM_VERSION
  script:
    - cd terraform/environments/staging/
    - terraform init
    - terraform apply -auto-approve
    - terraform output -json > ../../outputs/staging.json
  environment:
    name: staging
    url: https://staging-mongodb.example.com
  only:
    - develop

# Deploy to production (manual approval)
deploy:production:
  stage: deploy-production
  image: hashicorp/terraform:$TERRAFORM_VERSION
  script:
    - cd terraform/environments/production/
    - terraform init
    - terraform apply -auto-approve
    - |
      # Wait for cluster to be ready
      for i in {1..30}; do
        if mongosh "$MONGODB_URI" --eval "db.adminCommand('ping')" > /dev/null 2>&1; then
          echo "Cluster is ready"
          break
        fi
        echo "Waiting for cluster... ($i/30)"
        sleep 10
      done
    - # Run smoke tests
    - mongosh "$MONGODB_URI" --eval "db.adminCommand({buildInfo: 1})"
  environment:
    name: production
    url: https://mongodb.example.com
  when: manual
  only:
    - main

# Post-deployment validation
validate:production:
  stage: deploy-production
  image: mongo:7.0
  script:
    - |
      # Validate replica set status
      mongosh "$MONGODB_URI" --eval "
        const status = rs.status();
        assert(status.ok === 1, 'Replica set status check failed');
        const primary = status.members.filter(m => m.stateStr === 'PRIMARY');
        assert(primary.length === 1, 'Expected exactly one PRIMARY');
        const healthy = status.members.filter(m => m.health === 1);
        assert(healthy.length === status.members.length, 'Not all members are healthy');
      "
    - |
      # Validate indexes
      mongosh "$MONGODB_URI/production_db" --eval "
        const collections = db.getCollectionNames();
        collections.forEach(coll => {
          const indexes = db[coll].getIndexes();
          print('Collection: ' + coll + ', Indexes: ' + indexes.length);
        });
      "
  dependencies:
    - deploy:production
  only:
    - main
```

---

## Gestion des Secrets

### Vault Integration pour MongoDB

**Architecture Vault + MongoDB :**

```
┌──────────────────────────────────────────────────────────────┐
│                    HashiCorp Vault                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         MongoDB Database Secrets Engine                │  │
│  │                                                        │  │
│  │  • Dynamic credentials generation                      │  │
│  │  • Automatic rotation                                  │  │
│  │  • Lease management                                    │  │
│  │  • Audit logging                                       │  │
│  └────────────────────────────────────────────────────────┘  │
│                          │                                   │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Policy & Authentication                   │  │
│  │                                                        │  │
│  │  • Kubernetes auth                                     │  │
│  │  • AppRole                                             │  │
│  │  • LDAP / Active Directory                             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    Applications                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │  Pod / App  │    │  Pod / App  │    │  Pod / App  │       │
│  │             │    │             │    │             │       │
│  │  1. Auth to │    │  2. Request │    │  3. Use     │       │
│  │     Vault   │───▶│     creds   │───▶│     MongoDB │       │
│  │             │    │             │    │             │       │
│  └─────────────┘    └─────────────┘    └─────────────┘       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Configuration Vault pour MongoDB :**

```bash
# Enable MongoDB secrets engine
vault secrets enable database

# Configure MongoDB connection
vault write database/config/production-mongodb \
  plugin_name=mongodb-database-plugin \
  allowed_roles="app-role,admin-role" \
  connection_url="mongodb://{{username}}:{{password}}@mongodb.production.svc.cluster.local:27017/admin?replicaSet=rs0" \
  username="vault-admin" \
  password="$VAULT_ADMIN_PASSWORD"

# Create role for application
vault write database/roles/app-role \
  db_name=production-mongodb \
  creation_statements='{ "db": "app_database", "roles": [{ "role": "readWrite" }] }' \
  default_ttl="1h" \
  max_ttl="24h"

# Create role for admin
vault write database/roles/admin-role \
  db_name=production-mongodb \
  creation_statements='{ "db": "admin", "roles": [{ "role": "dbAdminAnyDatabase" }] }' \
  default_ttl="30m" \
  max_ttl="2h"
```

**Kubernetes Integration avec Vault :**

```yaml
# Service Account for Vault
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: mongodb
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: mongodb
---
# Application Deployment with Vault sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "app-role"
        vault.hashicorp.com/agent-inject-secret-mongodb-creds: "database/creds/app-role"
        vault.hashicorp.com/agent-inject-template-mongodb-creds: |
          {{- with secret "database/creds/app-role" -}}
          export MONGODB_USERNAME="{{ .Data.username }}"
          export MONGODB_PASSWORD="{{ .Data.password }}"
          export MONGODB_URI="mongodb://{{ .Data.username }}:{{ .Data.password }}@mongodb.production.svc.cluster.local:27017/app_database?replicaSet=rs0"
          {{- end }}
    spec:
      serviceAccountName: vault-auth
      containers:
      - name: app
        image: myapp:latest
        command: ["/bin/sh"]
        args:
          - -c
          - source /vault/secrets/mongodb-creds && ./start-app.sh
```

**Automatic Credential Rotation :**

```go
// Go application with Vault integration
package main

import (
    "context"
    "log"
    "time"

    vault "github.com/hashicorp/vault/api"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type MongoDBClient struct {
    client      *mongo.Client
    vaultClient *vault.Client
    lease       *vault.Secret
}

func NewMongoDBClient(vaultAddr, roleName string) (*MongoDBClient, error) {
    // Initialize Vault client
    vaultClient, err := vault.NewClient(&vault.Config{
        Address: vaultAddr,
    })
    if err != nil {
        return nil, err
    }

    // Kubernetes auth
    k8sToken, _ := ioutil.ReadFile("/var/run/secrets/kubernetes.io/serviceaccount/token")
    authData := map[string]interface{}{
        "role": roleName,
        "jwt":  string(k8sToken),
    }
    secret, err := vaultClient.Logical().Write("auth/kubernetes/login", authData)
    if err != nil {
        return nil, err
    }
    vaultClient.SetToken(secret.Auth.ClientToken)

    // Get MongoDB credentials
    lease, err := vaultClient.Logical().Read("database/creds/app-role")
    if err != nil {
        return nil, err
    }

    username := lease.Data["username"].(string)
    password := lease.Data["password"].(string)

    // Connect to MongoDB
    uri := fmt.Sprintf("mongodb://%s:%s@mongodb:27017/app_database?replicaSet=rs0",
        username, password)

    mongoClient, err := mongo.Connect(context.Background(),
        options.Client().ApplyURI(uri))
    if err != nil {
        return nil, err
    }

    mc := &MongoDBClient{
        client:      mongoClient,
        vaultClient: vaultClient,
        lease:       lease,
    }

    // Start credential renewal goroutine
    go mc.renewCredentials()

    return mc, nil
}

func (mc *MongoDBClient) renewCredentials() {
    renewer, err := mc.vaultClient.NewRenewer(&vault.RenewerInput{
        Secret: mc.lease,
    })
    if err != nil {
        log.Fatal(err)
    }

    go renewer.Renew()
    defer renewer.Stop()

    for {
        select {
        case err := <-renewer.DoneCh():
            if err != nil {
                log.Printf("Credential renewal error: %v", err)
                // Reconnect with new credentials
                mc.reconnect()
            }
        case renewal := <-renewer.RenewCh():
            log.Printf("Credentials renewed: %s", renewal.LeaseID)
        }
    }
}

func (mc *MongoDBClient) reconnect() {
    // Get new credentials
    lease, err := mc.vaultClient.Logical().Read("database/creds/app-role")
    if err != nil {
        log.Fatal(err)
    }

    username := lease.Data["username"].(string)
    password := lease.Data["password"].(string)

    // Disconnect old client
    mc.client.Disconnect(context.Background())

    // Connect with new credentials
    uri := fmt.Sprintf("mongodb://%s:%s@mongodb:27017/app_database?replicaSet=rs0",
        username, password)

    mongoClient, err := mongo.Connect(context.Background(),
        options.Client().ApplyURI(uri))
    if err != nil {
        log.Fatal(err)
    }

    mc.client = mongoClient
    mc.lease = lease
}
```

---

## Monitoring et Observabilité

### Stack de Monitoring

```yaml
# Prometheus ServiceMonitor for MongoDB
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mongodb-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: mongodb-exporter
  endpoints:
  - port: metrics
    interval: 30s
    scrapeTimeout: 10s
    path: /metrics
---
# MongoDB Exporter Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb-exporter
  template:
    metadata:
      labels:
        app: mongodb-exporter
    spec:
      containers:
      - name: mongodb-exporter
        image: percona/mongodb_exporter:0.40
        args:
          - --mongodb.uri=mongodb://exporter:$(MONGODB_EXPORTER_PASSWORD)@mongodb.production.svc.cluster.local:27017
          - --collect-all
          - --compatible-mode
        ports:
        - name: metrics
          containerPort: 9216
        env:
        - name: MONGODB_EXPORTER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-exporter-secret
              key: password
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**Grafana Dashboard :**

```json
{
  "dashboard": {
    "title": "MongoDB Cluster Overview",
    "panels": [
      {
        "title": "Replica Set Status",
        "targets": [
          {
            "expr": "mongodb_mongod_replset_member_state",
            "legendFormat": "{{member}}"
          }
        ]
      },
      {
        "title": "Operations per Second",
        "targets": [
          {
            "expr": "rate(mongodb_op_counters_total[5m])",
            "legendFormat": "{{type}}"
          }
        ]
      },
      {
        "title": "Connections",
        "targets": [
          {
            "expr": "mongodb_connections{state='current'}",
            "legendFormat": "Current"
          },
          {
            "expr": "mongodb_connections{state='available'}",
            "legendFormat": "Available"
          }
        ]
      },
      {
        "title": "Replication Lag",
        "targets": [
          {
            "expr": "mongodb_mongod_replset_member_replication_lag",
            "legendFormat": "{{member}}"
          }
        ]
      },
      {
        "title": "Query Execution Time",
        "targets": [
          {
            "expr": "rate(mongodb_mongod_metrics_query_executor_total[5m])",
            "legendFormat": "{{type}}"
          }
        ]
      },
      {
        "title": "Document Operations",
        "targets": [
          {
            "expr": "rate(mongodb_mongod_metrics_document_total[5m])",
            "legendFormat": "{{state}}"
          }
        ]
      }
    ]
  }
}
```

---

## Conclusion du Chapitre

Ce chapitre a présenté une vue d'ensemble complète du DevOps et du déploiement pour MongoDB, couvrant :

1. **Infrastructure as Code** : Provisionnement automatisé avec Terraform, Ansible
2. **Orchestration** : Déploiement sur Kubernetes avec Operators
3. **Patterns d'architecture** : Replica Sets, Sharding, Multi-région
4. **CI/CD** : Pipelines automatisés pour déploiement et validation
5. **Gestion des secrets** : Intégration Vault pour sécurité renforcée
6. **Monitoring** : Observabilité avec Prometheus et Grafana

Les sections suivantes détailleront chaque aspect avec des exemples pratiques approfondis et des configurations de production réelles.

---


⏭️ [Infrastructure as Code pour MongoDB](/18-devops-deploiement/01-infrastructure-as-code.md)
