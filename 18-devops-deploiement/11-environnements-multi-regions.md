🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.11 Environnements Multi-Régions

## Introduction

Les déploiements multi-régions sont essentiels pour les applications globales nécessitant haute disponibilité, faible latence et résilience aux pannes régionales. Pour MongoDB, cela implique une architecture distribuée complexe avec réplication cross-region, failover automatique, et gestion de la cohérence des données.

```
┌───────────────────────────────────────────────────────────────────┐
│              Global Multi-Region Architecture                     │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                   Global Traffic Manager                    │  │
│  │            (Route53 / Cloud CDN / Cloudflare)               │  │
│  │                                                             │  │
│  │  • Geo-routing                                              │  │
│  │  • Health checks                                            │  │
│  │  • Failover policies                                        │  │
│  │  • Latency-based routing                                    │  │
│  └────────┬───────────────────────┬────────────────────┬───────┘  │
│           │                       │                    │          │
│           ▼                       ▼                    ▼          │
│  ┌──────────────────┐    ┌─────────────────┐  ┌────────────────┐  │
│  │   Region: US     │    │  Region: EU     │  │ Region: APAC   │  │
│  │   us-east-1      │    │  eu-west-1      │  │ ap-southeast-1 │  │
│  │                  │    │                 │  │                │  │
│  │  ┌────────────┐  │    │  ┌────────────┐ │  │ ┌────────────┐ │  │
│  │  │ K8s Cluster│  │    │  │ K8s Cluster│ │  │ │ K8s Cluster│ │  │
│  │  │            │  │    │  │            │ │  │ │            │ │  │
│  │  │ ┌────────┐ │  │    │  │ ┌────────┐ │ │  │ │ ┌────────┐ │ │  │
│  │  │ │  Apps  │ │  │    │  │ │  Apps  │ │ │  │ │ │  Apps  │ │ │  │
│  │  │ │ (3pod) │ │  │    │  │ │ (3pod) │ │ │  │ │ │ (3pod) │ │ │  │
│  │  │ └────────┘ │  │    │  │ └────────┘ │ │  │ │ └────────┘ │ │  │
│  │  └────────────┘  │    │  └────────────┘ │  │ └────────────┘ │  │
│  │        │         │    │        │        │  │        │       │  │
│  │        ▼         │    │        ▼        │  │        ▼       │  │
│  │  ┌────────────┐  │    │  ┌────────────┐ │  │ ┌────────────┐ │  │
│  │  │  MongoDB   │◄─┼────┼─►│  MongoDB   │◄┼──┼►│  MongoDB   │ │  │
│  │  │  Replica   │  │    │  │  Replica   │ │  │ │  Replica   │ │  │
│  │  │   Set      │  │    │  │   Set      │ │  │ │   Set      │ │  │
│  │  │            │  │    │  │            │ │  │ │            │ │  │
│  │  │ P─S─S      │  │    │  │ S─P─S      │ │  │ │ S─S─P      │ │  │
│  │  │ (priority) │  │    │  │ (priority) │ │  │ │ (priority) │ │  │
│  │  └────────────┘  │    │  └────────────┘ │  │ └────────────┘ │  │
│  │                  │    │                 │  │                │  │
│  │  ┌────────────┐  │    │  ┌────────────┐ │  │ ┌────────────┐ │  │
│  │  │  Backup    │  │    │  │  Backup    │ │  │ │  Backup    │ │  │
│  │  │  (S3/GCS)  │  │    │  │  (S3/GCS)  │ │  │ │  (S3/GCS)  │ │  │
│  │  └────────────┘  │    │  └────────────┘ │  │ └────────────┘ │  │
│  └──────────────────┘    └─────────────────┘  └────────────────┘  │
│                                                                   │
│  Key Features:                                                    │
│  • Geographic distribution                                        │
│  • Local read/write latency optimization                          │
│  • Automatic failover across regions                              │
│  • Data sovereignty compliance                                    │
│  • 99.999% availability target                                    │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Objectifs Multi-Régions

```yaml
# multi-region-objectives.yaml
---
availability:
  target: "99.999%"  # Five nines
  annual_downtime: "5.26 minutes"
  strategies:
    - "Multi-region active-active"
    - "Automatic failover"
    - "Health monitoring"
    - "Circuit breakers"

performance:
  latency_targets:
    read_local: "< 10ms"
    read_cross_region: "< 100ms"
    write_primary: "< 50ms"
  strategies:
    - "Geographic proximity routing"
    - "Read from nearest replica"
    - "Write to regional primary"
    - "CDN for static content"

disaster_recovery:
  rpo: "< 1 minute"  # Recovery Point Objective
  rto: "< 5 minutes" # Recovery Time Objective
  strategies:
    - "Continuous replication"
    - "Point-in-time recovery"
    - "Cross-region backups"
    - "Automated failover"

compliance:
  requirements:
    - "GDPR (EU data in EU)"
    - "Data residency laws"
    - "Audit logging"
    - "Encryption in transit"
  strategies:
    - "Regional data isolation"
    - "Sharding by geography"
    - "Compliance monitoring"

cost_optimization:
  considerations:
    - "Network egress costs"
    - "Data transfer pricing"
    - "Regional pricing differences"
    - "Reserved capacity"
  strategies:
    - "Read preference optimization"
    - "Compression"
    - "Smart routing"
    - "Capacity planning"
```

---

## Architecture MongoDB Multi-Régions

### Replica Set Distribué

```yaml
# mongodb-multi-region-replicaset.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-config
  namespace: mongodb
data:
  mongodb.conf: |
    # Multi-region replica set configuration

    net:
      port: 27017
      bindIp: 0.0.0.0
      maxIncomingConnections: 65536

      # Compression for cross-region traffic
      compression:
        compressors: snappy,zstd

      tls:
        mode: requireTLS
        certificateKeyFile: /etc/mongodb/tls/mongodb.pem
        CAFile: /etc/mongodb/tls/ca.crt

    storage:
      dbPath: /data/db
      engine: wiredTiger
      wiredTiger:
        engineConfig:
          cacheSizeGB: 16
          journalCompressor: snappy
        collectionConfig:
          blockCompressor: snappy

    replication:
      replSetName: global-rs
      oplogSizeMB: 51200  # 50GB for multi-region

      # Enable majority read concern
      enableMajorityReadConcern: true

    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 100

    setParameter:
      # Increase heartbeat for cross-region
      electionTimeoutMillis: 10000
      heartbeatIntervalMillis: 2000
      heartbeatTimeoutSecs: 10

      # Enable causal consistency
      enableCausalConsistency: true
---
# StatefulSet for US Region
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-us
  namespace: mongodb
spec:
  serviceName: mongodb-us-headless
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
      region: us

  template:
    metadata:
      labels:
        app: mongodb
        region: us

    spec:
      affinity:
        # Anti-affinity to spread across zones
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - mongodb
              topologyKey: topology.kubernetes.io/zone

      containers:
        - name: mongodb
          image: mongo:7.0

          command:
            - mongod
            - --config=/etc/mongodb/mongodb.conf
            - --replSet=global-rs

          ports:
            - name: mongodb
              containerPort: 27017

          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: root-username

            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: root-password

          volumeMounts:
            - name: data
              mountPath: /data/db
            - name: config
              mountPath: /etc/mongodb
            - name: tls
              mountPath: /etc/mongodb/tls

          resources:
            requests:
              memory: "32Gi"
              cpu: "8000m"
            limits:
              memory: "64Gi"
              cpu: "16000m"

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
                - "rs.status()"
            initialDelaySeconds: 30
            periodSeconds: 10

      volumes:
        - name: config
          configMap:
            name: mongodb-config
        - name: tls
          secret:
            secretName: mongodb-tls

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 1Ti
```

### Replica Set Initialization Script

```javascript
// scripts/init-global-replica-set.js
/**
 * Initialize global multi-region replica set
 * Run from primary region (US)
 */

const config = {
  _id: "global-rs",
  version: 1,

  members: [
    // US Region (Primary preferred)
    {
      _id: 0,
      host: "mongodb-us-0.mongodb-us-headless.mongodb.svc.cluster.local:27017",
      priority: 10,
      tags: { region: "us", datacenter: "us-east-1a" }
    },
    {
      _id: 1,
      host: "mongodb-us-1.mongodb-us-headless.mongodb.svc.cluster.local:27017",
      priority: 9,
      tags: { region: "us", datacenter: "us-east-1b" }
    },
    {
      _id: 2,
      host: "mongodb-us-2.mongodb-us-headless.mongodb.svc.cluster.local:27017",
      priority: 8,
      tags: { region: "us", datacenter: "us-east-1c" }
    },

    // EU Region (Secondary with high priority)
    {
      _id: 3,
      host: "mongodb-eu-0.mongodb-eu-headless.mongodb.svc.cluster.local:27017",
      priority: 7,
      tags: { region: "eu", datacenter: "eu-west-1a" }
    },
    {
      _id: 4,
      host: "mongodb-eu-1.mongodb-eu-headless.mongodb.svc.cluster.local:27017",
      priority: 6,
      tags: { region: "eu", datacenter: "eu-west-1b" }
    },
    {
      _id: 5,
      host: "mongodb-eu-2.mongodb-eu-headless.mongodb.svc.cluster.local:27017",
      priority: 5,
      tags: { region: "eu", datacenter: "eu-west-1c" }
    },

    // APAC Region (Secondary with lower priority)
    {
      _id: 6,
      host: "mongodb-apac-0.mongodb-apac-headless.mongodb.svc.cluster.local:27017",
      priority: 4,
      tags: { region: "apac", datacenter: "ap-southeast-1a" }
    },
    {
      _id: 7,
      host: "mongodb-apac-1.mongodb-apac-headless.mongodb.svc.cluster.local:27017",
      priority: 3,
      tags: { region: "apac", datacenter: "ap-southeast-1b" }
    },
    {
      _id: 8,
      host: "mongodb-apac-2.mongodb-apac-headless.mongodb.svc.cluster.local:27017",
      priority: 2,
      tags: { region: "apac", datacenter: "ap-southeast-1c" }
    }
  ],

  settings: {
    // Increase election timeout for cross-region
    electionTimeoutMillis: 10000,

    // Heartbeat configuration
    heartbeatIntervalMillis: 2000,
    heartbeatTimeoutSecs: 10,

    // Write concern for majority
    getLastErrorDefaults: {
      w: "majority",
      wtimeout: 5000
    },

    // Chaining allowed for better cross-region replication
    chainingAllowed: true
  }
};

// Initialize replica set
rs.initiate(config);

// Wait for initialization
sleep(5000);

// Verify status
print("Replica Set Status:");
printjson(rs.status());

// Configure read preference tags
print("\nConfiguring read preference tags...");
rs.conf();
```

---

## Terraform Multi-Régions

### Provider Configuration

```hcl
# terraform/main.tf
# Multi-region infrastructure

terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }

  backend "s3" {
    bucket         = "terraform-state-global"
    key            = "multi-region/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Provider aliases for multiple regions
provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "apac"
  region = "ap-southeast-1"
}

# Kubernetes providers for each region
provider "kubernetes" {
  alias = "us"

  host                   = module.eks_us.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks_us.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      module.eks_us.cluster_name,
      "--region",
      "us-east-1"
    ]
  }
}

provider "kubernetes" {
  alias = "eu"

  host                   = module.eks_eu.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks_eu.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      module.eks_eu.cluster_name,
      "--region",
      "eu-west-1"
    ]
  }
}

provider "kubernetes" {
  alias = "apac"

  host                   = module.eks_apac.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks_apac.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      module.eks_apac.cluster_name,
      "--region",
      "ap-southeast-1"
    ]
  }
}
```

### VPC Peering Multi-Régions

```hcl
# terraform/vpc-peering.tf
# VPC Peering for cross-region MongoDB replication

# US VPC
module "vpc_us" {
  source = "./modules/vpc"

  providers = {
    aws = aws.us
  }

  name = "mongodb-vpc-us"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Region = "us"
  }
}

# EU VPC
module "vpc_eu" {
  source = "./modules/vpc"

  providers = {
    aws = aws.eu
  }

  name = "mongodb-vpc-eu"
  cidr = "10.1.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  private_subnets = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
  public_subnets  = ["10.1.101.0/24", "10.1.102.0/24", "10.1.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Region = "eu"
  }
}

# APAC VPC
module "vpc_apac" {
  source = "./modules/vpc"

  providers = {
    aws = aws.apac
  }

  name = "mongodb-vpc-apac"
  cidr = "10.2.0.0/16"

  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
  public_subnets  = ["10.2.101.0/24", "10.2.102.0/24", "10.2.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Region = "apac"
  }
}

# VPC Peering: US <-> EU
resource "aws_vpc_peering_connection" "us_to_eu" {
  provider = aws.us

  vpc_id        = module.vpc_us.vpc_id
  peer_vpc_id   = module.vpc_eu.vpc_id
  peer_region   = "eu-west-1"
  auto_accept   = false

  tags = {
    Name = "us-to-eu-peering"
  }
}

resource "aws_vpc_peering_connection_accepter" "eu_accept_us" {
  provider = aws.eu

  vpc_peering_connection_id = aws_vpc_peering_connection.us_to_eu.id
  auto_accept               = true

  tags = {
    Name = "eu-accept-us-peering"
  }
}

# VPC Peering: US <-> APAC
resource "aws_vpc_peering_connection" "us_to_apac" {
  provider = aws.us

  vpc_id        = module.vpc_us.vpc_id
  peer_vpc_id   = module.vpc_apac.vpc_id
  peer_region   = "ap-southeast-1"
  auto_accept   = false

  tags = {
    Name = "us-to-apac-peering"
  }
}

resource "aws_vpc_peering_connection_accepter" "apac_accept_us" {
  provider = aws.apac

  vpc_peering_connection_id = aws_vpc_peering_connection.us_to_apac.id
  auto_accept               = true

  tags = {
    Name = "apac-accept-us-peering"
  }
}

# VPC Peering: EU <-> APAC
resource "aws_vpc_peering_connection" "eu_to_apac" {
  provider = aws.eu

  vpc_id        = module.vpc_eu.vpc_id
  peer_vpc_id   = module.vpc_apac.vpc_id
  peer_region   = "ap-southeast-1"
  auto_accept   = false

  tags = {
    Name = "eu-to-apac-peering"
  }
}

resource "aws_vpc_peering_connection_accepter" "apac_accept_eu" {
  provider = aws.apac

  vpc_peering_connection_id = aws_vpc_peering_connection.eu_to_apac.id
  auto_accept               = true

  tags = {
    Name = "apac-accept-eu-peering"
  }
}

# Route tables for peering - US side
resource "aws_route" "us_to_eu" {
  provider = aws.us

  count = length(module.vpc_us.private_route_table_ids)

  route_table_id            = module.vpc_us.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_eu.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.us_to_eu.id
}

resource "aws_route" "us_to_apac" {
  provider = aws.us

  count = length(module.vpc_us.private_route_table_ids)

  route_table_id            = module.vpc_us.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_apac.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.us_to_apac.id
}

# Route tables for peering - EU side
resource "aws_route" "eu_to_us" {
  provider = aws.eu

  count = length(module.vpc_eu.private_route_table_ids)

  route_table_id            = module.vpc_eu.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_us.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.us_to_eu.id
}

resource "aws_route" "eu_to_apac" {
  provider = aws.eu

  count = length(module.vpc_eu.private_route_table_ids)

  route_table_id            = module.vpc_eu.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_apac.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.eu_to_apac.id
}

# Route tables for peering - APAC side
resource "aws_route" "apac_to_us" {
  provider = aws.apac

  count = length(module.vpc_apac.private_route_table_ids)

  route_table_id            = module.vpc_apac.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_us.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.us_to_apac.id
}

resource "aws_route" "apac_to_eu" {
  provider = aws.apac

  count = length(module.vpc_apac.private_route_table_ids)

  route_table_id            = module.vpc_apac.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_eu.vpc_cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.eu_to_apac.id
}
```

### EKS Clusters Multi-Régions

```hcl
# terraform/eks-clusters.tf
# EKS clusters in multiple regions

# US Cluster
module "eks_us" {
  source = "./modules/eks"

  providers = {
    aws = aws.us
  }

  cluster_name    = "mongodb-cluster-us"
  cluster_version = "1.28"

  vpc_id     = module.vpc_us.vpc_id
  subnet_ids = module.vpc_us.private_subnets

  node_groups = {
    mongodb = {
      desired_size = 3
      min_size     = 3
      max_size     = 9

      instance_types = ["m6i.4xlarge"]

      labels = {
        role   = "mongodb"
        region = "us"
      }

      taints = [
        {
          key    = "dedicated"
          value  = "mongodb"
          effect = "NoSchedule"
        }
      ]
    }

    application = {
      desired_size = 6
      min_size     = 3
      max_size     = 20

      instance_types = ["m6i.2xlarge"]

      labels = {
        role   = "application"
        region = "us"
      }
    }
  }

  tags = {
    Region = "us"
  }
}

# EU Cluster
module "eks_eu" {
  source = "./modules/eks"

  providers = {
    aws = aws.eu
  }

  cluster_name    = "mongodb-cluster-eu"
  cluster_version = "1.28"

  vpc_id     = module.vpc_eu.vpc_id
  subnet_ids = module.vpc_eu.private_subnets

  node_groups = {
    mongodb = {
      desired_size = 3
      min_size     = 3
      max_size     = 9

      instance_types = ["m6i.4xlarge"]

      labels = {
        role   = "mongodb"
        region = "eu"
      }

      taints = [
        {
          key    = "dedicated"
          value  = "mongodb"
          effect = "NoSchedule"
        }
      ]
    }

    application = {
      desired_size = 6
      min_size     = 3
      max_size     = 20

      instance_types = ["m6i.2xlarge"]

      labels = {
        role   = "application"
        region = "eu"
      }
    }
  }

  tags = {
    Region = "eu"
  }
}

# APAC Cluster
module "eks_apac" {
  source = "./modules/eks"

  providers = {
    aws = aws.apac
  }

  cluster_name    = "mongodb-cluster-apac"
  cluster_version = "1.28"

  vpc_id     = module.vpc_apac.vpc_id
  subnet_ids = module.vpc_apac.private_subnets

  node_groups = {
    mongodb = {
      desired_size = 3
      min_size     = 3
      max_size     = 9

      instance_types = ["m6i.4xlarge"]

      labels = {
        role   = "mongodb"
        region = "apac"
      }

      taints = [
        {
          key    = "dedicated"
          value  = "mongodb"
          effect = "NoSchedule"
        }
      ]
    }

    application = {
      desired_size = 6
      min_size     = 3
      max_size     = 20

      instance_types = ["m6i.2xlarge"]

      labels = {
        role   = "application"
        region = "apac"
      }
    }
  }

  tags = {
    Region = "apac"
  }
}
```

---

## Global Load Balancing

### Route53 Configuration

```hcl
# terraform/route53.tf
# Global traffic routing with Route53

# Hosted zone
resource "aws_route53_zone" "main" {
  provider = aws.us

  name = "myapp.example.com"

  tags = {
    Name = "MyApp Global Zone"
  }
}

# Health checks for each region
resource "aws_route53_health_check" "us" {
  provider = aws.us

  fqdn              = "api-us.myapp.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name   = "US Health Check"
    Region = "us"
  }
}

resource "aws_route53_health_check" "eu" {
  provider = aws.us

  fqdn              = "api-eu.myapp.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name   = "EU Health Check"
    Region = "eu"
  }
}

resource "aws_route53_health_check" "apac" {
  provider = aws.us

  fqdn              = "api-apac.myapp.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name   = "APAC Health Check"
    Region = "apac"
  }
}

# Geolocation routing for API
resource "aws_route53_record" "api_us" {
  provider = aws.us

  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.example.com"
  type    = "A"

  geolocation_routing_policy {
    continent = "NA"  # North America
  }

  set_identifier = "us"
  health_check_id = aws_route53_health_check.us.id

  alias {
    name                   = module.alb_us.dns_name
    zone_id                = module.alb_us.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_eu" {
  provider = aws.us

  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.example.com"
  type    = "A"

  geolocation_routing_policy {
    continent = "EU"  # Europe
  }

  set_identifier = "eu"
  health_check_id = aws_route53_health_check.eu.id

  alias {
    name                   = module.alb_eu.dns_name
    zone_id                = module.alb_eu.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_apac" {
  provider = aws.us

  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.example.com"
  type    = "A"

  geolocation_routing_policy {
    continent = "AS"  # Asia
  }

  set_identifier = "apac"
  health_check_id = aws_route53_health_check.apac.id

  alias {
    name                   = module.alb_apac.dns_name
    zone_id                = module.alb_apac.zone_id
    evaluate_target_health = true
  }
}

# Failover: Default route (latency-based)
resource "aws_route53_record" "api_default" {
  provider = aws.us

  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.example.com"
  type    = "A"

  geolocation_routing_policy {
    continent = "*"  # Default
  }

  set_identifier = "default"

  # Route to closest healthy region
  latency_routing_policy {
    region = "us-east-1"
  }

  health_check_id = aws_route53_health_check.us.id

  alias {
    name                   = module.alb_us.dns_name
    zone_id                = module.alb_us.zone_id
    evaluate_target_health = true
  }
}

# CloudWatch alarms for health checks
resource "aws_cloudwatch_metric_alarm" "us_health" {
  provider = aws.us

  alarm_name          = "route53-health-us"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthCheckStatus"
  namespace           = "AWS/Route53"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1
  alarm_description   = "US region health check failed"

  dimensions = {
    HealthCheckId = aws_route53_health_check.us.id
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## Application Configuration Multi-Régions

### MongoDB Connection avec Read Preference

```javascript
// config/mongodb-multi-region.js
const { MongoClient, ReadPreference } = require('mongodb');

class MultiRegionMongoClient {
  constructor() {
    this.client = null;
    this.currentRegion = process.env.AWS_REGION || 'us-east-1';

    // Connection string avec tous les membres du replica set
    this.uri = process.env.MONGODB_URI ||
      'mongodb://mongodb-us-0.mongodb-us-headless:27017,' +
      'mongodb-us-1.mongodb-us-headless:27017,' +
      'mongodb-us-2.mongodb-us-headless:27017,' +
      'mongodb-eu-0.mongodb-eu-headless:27017,' +
      'mongodb-eu-1.mongodb-eu-headless:27017,' +
      'mongodb-eu-2.mongodb-eu-headless:27017,' +
      'mongodb-apac-0.mongodb-apac-headless:27017,' +
      'mongodb-apac-1.mongodb-apac-headless:27017,' +
      'mongodb-apac-2.mongodb-apac-headless:27017' +
      '/myapp?replicaSet=global-rs&authSource=admin';

    // Map regions to read preference tags
    this.regionTags = {
      'us-east-1': { region: 'us' },
      'eu-west-1': { region: 'eu' },
      'ap-southeast-1': { region: 'apac' }
    };
  }

  async connect() {
    const options = {
      maxPoolSize: 100,
      minPoolSize: 10,
      maxIdleTimeMS: 60000,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,

      // Compression for cross-region traffic
      compressors: ['snappy', 'zlib'],

      // Retry configuration
      retryWrites: true,
      retryReads: true,

      // Read preference: nearest in current region
      readPreference: ReadPreference.NEAREST,
      readPreferenceTags: [
        this.regionTags[this.currentRegion],  // Prefer local region
        {}  // Fallback to any
      ],

      // Write concern
      writeConcern: {
        w: 'majority',
        wtimeout: 5000,
        journal: true
      },

      // Read concern
      readConcern: {
        level: 'majority'
      }
    };

    this.client = await MongoClient.connect(this.uri, options);
    console.log(`Connected to MongoDB global cluster from region: ${this.currentRegion}`);

    // Monitor topology changes
    this.client.on('topologyDescriptionChanged', (event) => {
      console.log('Topology changed:', {
        newDescription: event.newDescription.type,
        previousDescription: event.previousDescription.type
      });
    });

    return this.client;
  }

  getDb(dbName = 'myapp') {
    if (!this.client) {
      throw new Error('Client not connected');
    }
    return this.client.db(dbName);
  }

  // Get collection with region-aware read preference
  getCollection(collectionName, options = {}) {
    const db = this.getDb();

    // Override read preference if specified
    if (options.readPreference) {
      const readPreference = new ReadPreference(
        options.readPreference,
        [this.regionTags[this.currentRegion], {}]
      );

      return db.collection(collectionName, { readPreference });
    }

    return db.collection(collectionName);
  }

  // Write to primary (always)
  async writeOperation(collectionName, operation, data) {
    const collection = this.getCollection(collectionName, {
      readPreference: ReadPreference.PRIMARY
    });

    const session = this.client.startSession();

    try {
      return await session.withTransaction(async () => {
        switch (operation) {
          case 'insertOne':
            return await collection.insertOne(data, { session });
          case 'updateOne':
            return await collection.updateOne(
              data.filter,
              data.update,
              { session }
            );
          case 'deleteOne':
            return await collection.deleteOne(data.filter, { session });
          default:
            throw new Error(`Unknown operation: ${operation}`);
        }
      });
    } finally {
      await session.endSession();
    }
  }

  // Read from nearest in region
  async readOperation(collectionName, operation, query) {
    const collection = this.getCollection(collectionName, {
      readPreference: ReadPreference.NEAREST
    });

    switch (operation) {
      case 'findOne':
        return await collection.findOne(query);
      case 'find':
        return await collection.find(query).toArray();
      case 'aggregate':
        return await collection.aggregate(query).toArray();
      default:
        throw new Error(`Unknown operation: ${operation}`);
    }
  }

  async close() {
    if (this.client) {
      await this.client.close();
      console.log('Disconnected from MongoDB');
    }
  }
}

module.exports = MultiRegionMongoClient;
```

### Application avec Zone Awareness

```javascript
// app.js
const express = require('express');
const MultiRegionMongoClient = require('./config/mongodb-multi-region');

const app = express();
app.use(express.json());

// Initialize MongoDB client
const mongoClient = new MultiRegionMongoClient();

// Middleware pour ajouter le header de région
app.use((req, res, next) => {
  res.setHeader('X-Region', process.env.AWS_REGION || 'unknown');
  next();
});

// Health check endpoint
app.get('/health', async (req, res) => {
  try {
    const db = mongoClient.getDb();
    await db.admin().ping();

    res.json({
      status: 'healthy',
      region: process.env.AWS_REGION,
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
      region: process.env.AWS_REGION
    });
  }
});

// Get user (read from nearest)
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await mongoClient.readOperation(
      'users',
      'findOne',
      { _id: req.params.id }
    );

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({
      user,
      servedFrom: process.env.AWS_REGION
    });
  } catch (error) {
    console.error('Error fetching user:', error);
    res.status(500).json({ error: error.message });
  }
});

// Create user (write to primary)
app.post('/api/users', async (req, res) => {
  try {
    const result = await mongoClient.writeOperation(
      'users',
      'insertOne',
      {
        ...req.body,
        createdAt: new Date(),
        createdIn: process.env.AWS_REGION
      }
    );

    res.status(201).json({
      id: result.insertedId,
      region: process.env.AWS_REGION
    });
  } catch (error) {
    console.error('Error creating user:', error);
    res.status(500).json({ error: error.message });
  }
});

// Update user (write to primary)
app.put('/api/users/:id', async (req, res) => {
  try {
    const result = await mongoClient.writeOperation(
      'users',
      'updateOne',
      {
        filter: { _id: req.params.id },
        update: {
          $set: {
            ...req.body,
            updatedAt: new Date(),
            updatedIn: process.env.AWS_REGION
          }
        }
      }
    );

    if (result.matchedCount === 0) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json({
      modified: result.modifiedCount > 0,
      region: process.env.AWS_REGION
    });
  } catch (error) {
    console.error('Error updating user:', error);
    res.status(500).json({ error: error.message });
  }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing connections...');
  await mongoClient.close();
  process.exit(0);
});

// Start server
const PORT = process.env.PORT || 3000;

async function start() {
  try {
    await mongoClient.connect();

    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
      console.log(`Region: ${process.env.AWS_REGION}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

start();
```

---

## Disaster Recovery

### Backup Strategy Multi-Régions

```yaml
# backup-cronjob-multi-region.yaml
# Automated backups avec cross-region replication

apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup-us
  namespace: mongodb
spec:
  # Backup every 6 hours
  schedule: "0 */6 * * *"

  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3

  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: mongodb-backup

          containers:
            - name: backup
              image: mongo:7.0

              command:
                - /bin/bash
                - -c
                - |
                  set -e

                  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                  BACKUP_NAME="mongodb-backup-us-${TIMESTAMP}"
                  BACKUP_DIR="/backup/${BACKUP_NAME}"

                  echo "Starting backup: ${BACKUP_NAME}"

                  # Create backup with mongodump
                  mongodump \
                    --uri="${MONGODB_URI}" \
                    --gzip \
                    --archive="${BACKUP_DIR}.archive.gz" \
                    --oplog \
                    --numParallelCollections=4

                  echo "Backup completed, uploading to S3..."

                  # Upload to local region S3
                  aws s3 cp \
                    "${BACKUP_DIR}.archive.gz" \
                    "s3://mongodb-backups-us/${BACKUP_NAME}.archive.gz" \
                    --storage-class GLACIER_IR

                  # Replicate to other regions
                  echo "Replicating to EU..."
                  aws s3 cp \
                    "s3://mongodb-backups-us/${BACKUP_NAME}.archive.gz" \
                    "s3://mongodb-backups-eu/${BACKUP_NAME}.archive.gz" \
                    --region eu-west-1 \
                    --storage-class GLACIER_IR

                  echo "Replicating to APAC..."
                  aws s3 cp \
                    "s3://mongodb-backups-us/${BACKUP_NAME}.archive.gz" \
                    "s3://mongodb-backups-apac/${BACKUP_NAME}.archive.gz" \
                    --region ap-southeast-1 \
                    --storage-class GLACIER_IR

                  # Verify backups
                  echo "Verifying backups..."
                  for region in us eu apac; do
                    SIZE=$(aws s3 ls "s3://mongodb-backups-${region}/${BACKUP_NAME}.archive.gz" \
                      --region ${region} | awk '{print $3}')
                    echo "Backup in ${region}: ${SIZE} bytes"
                  done

                  echo "Backup and replication completed successfully"

                  # Cleanup old backups (keep last 30 days)
                  echo "Cleaning up old backups..."
                  aws s3 ls "s3://mongodb-backups-us/" \
                    | awk '{print $4}' \
                    | while read file; do
                      AGE=$(( $(date +%s) - $(date -d "$(echo $file | cut -d'-' -f3-4)" +%s) ))
                      if [ $AGE -gt 2592000 ]; then  # 30 days
                        echo "Deleting old backup: ${file}"
                        aws s3 rm "s3://mongodb-backups-us/${file}"
                      fi
                    done

              env:
                - name: MONGODB_URI
                  valueFrom:
                    secretKeyRef:
                      name: mongodb-secret
                      key: connection-string

                - name: AWS_REGION
                  value: "us-east-1"

              volumeMounts:
                - name: backup
                  mountPath: /backup

          volumes:
            - name: backup
              emptyDir:
                sizeLimit: 100Gi

          restartPolicy: OnFailure
```

### Point-in-Time Recovery

```bash
#!/bin/bash
# scripts/pitr-restore.sh
# Point-in-time recovery from backup

set -e

RESTORE_TIMESTAMP="${1:-$(date -d '1 hour ago' +%Y%m%d-%H%M%S)}"
SOURCE_REGION="${2:-us}"
TARGET_REGION="${3:-us}"

echo "=========================================="
echo "Point-in-Time Recovery"
echo "=========================================="
echo "Restore timestamp: ${RESTORE_TIMESTAMP}"
echo "Source region: ${SOURCE_REGION}"
echo "Target region: ${TARGET_REGION}"
echo "=========================================="

# Find closest backup before timestamp
echo "Finding closest backup..."
BACKUP_FILE=$(aws s3 ls "s3://mongodb-backups-${SOURCE_REGION}/" \
  | awk '{print $4}' \
  | sort -r \
  | awk -v ts="${RESTORE_TIMESTAMP}" '$0 < "mongodb-backup-'${SOURCE_REGION}'-"ts {print; exit}')

if [ -z "$BACKUP_FILE" ]; then
  echo "Error: No backup found before ${RESTORE_TIMESTAMP}"
  exit 1
fi

echo "Found backup: ${BACKUP_FILE}"

# Download backup
echo "Downloading backup..."
BACKUP_DIR="/tmp/mongodb-restore"
mkdir -p "${BACKUP_DIR}"

aws s3 cp \
  "s3://mongodb-backups-${SOURCE_REGION}/${BACKUP_FILE}" \
  "${BACKUP_DIR}/${BACKUP_FILE}"

echo "Backup downloaded"

# Extract backup timestamp
BACKUP_TIMESTAMP=$(echo ${BACKUP_FILE} | grep -oP '\d{8}-\d{6}')
echo "Backup timestamp: ${BACKUP_TIMESTAMP}"

# Stop application in target region
echo "Stopping application in ${TARGET_REGION}..."
kubectl scale deployment myapp \
  --replicas=0 \
  --namespace=production \
  --context="mongodb-cluster-${TARGET_REGION}"

# Wait for connections to drain
echo "Waiting for connections to drain..."
sleep 30

# Restore backup
echo "Restoring backup..."
mongorestore \
  --uri="${MONGODB_URI}" \
  --gzip \
  --archive="${BACKUP_DIR}/${BACKUP_FILE}" \
  --oplogReplay \
  --oplogLimit="${RESTORE_TIMESTAMP}" \
  --numParallelCollections=4 \
  --drop

echo "Backup restored"

# Verify restoration
echo "Verifying restoration..."
mongosh "${MONGODB_URI}" --eval "
  const latestDoc = db.audit_log.find().sort({timestamp: -1}).limit(1).toArray()[0];
  print('Latest document timestamp: ' + latestDoc.timestamp);

  if (new Date(latestDoc.timestamp) > new Date('${RESTORE_TIMESTAMP}')) {
    print('ERROR: Found documents newer than restore point');
    quit(1);
  }

  print('Verification passed');
"

# Restart application
echo "Restarting application..."
kubectl scale deployment myapp \
  --replicas=3 \
  --namespace=production \
  --context="mongodb-cluster-${TARGET_REGION}"

echo "=========================================="
echo "Point-in-Time Recovery completed"
echo "=========================================="
```

### Automated Failover Script

```bash
#!/bin/bash
# scripts/automated-failover.sh
# Automated failover to secondary region

set -e

PRIMARY_REGION="${1:-us}"
SECONDARY_REGION="${2:-eu}"

echo "=========================================="
echo "Automated Failover"
echo "=========================================="
echo "Primary region: ${PRIMARY_REGION}"
echo "Secondary region: ${SECONDARY_REGION}"
echo "=========================================="

# Check primary region health
echo "Checking primary region health..."
PRIMARY_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://api-${PRIMARY_REGION}.myapp.example.com/health" || echo "000")

if [ "$PRIMARY_HEALTH" = "200" ]; then
  echo "Primary region is healthy, no failover needed"
  exit 0
fi

echo "WARNING: Primary region is unhealthy (status: ${PRIMARY_HEALTH})"
echo "Initiating failover to ${SECONDARY_REGION}..."

# Verify secondary region health
echo "Checking secondary region health..."
SECONDARY_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://api-${SECONDARY_REGION}.myapp.example.com/health")

if [ "$SECONDARY_HEALTH" != "200" ]; then
  echo "ERROR: Secondary region is also unhealthy"
  # Send critical alert
  aws sns publish \
    --topic-arn "arn:aws:sns:us-east-1:123456789012:critical-alerts" \
    --message "CRITICAL: Both primary and secondary regions are unhealthy"
  exit 1
fi

echo "Secondary region is healthy, proceeding with failover..."

# Step 1: Update Route53 to point to secondary region
echo "Updating Route53..."
aws route53 change-resource-record-sets \
  --hosted-zone-id "${HOSTED_ZONE_ID}" \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.myapp.example.com",
          "Type": "A",
          "SetIdentifier": "failover",
          "Failover": "PRIMARY",
          "AliasTarget": {
            "HostedZoneId": "'${SECONDARY_ALB_ZONE_ID}'",
            "DNSName": "'${SECONDARY_ALB_DNS}'",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

echo "Route53 updated"

# Step 2: Reconfigure replica set to promote secondary
echo "Reconfiguring replica set..."
mongosh "${MONGODB_URI}" --eval "
  // Force primary to step down
  try {
    db.adminCommand({replSetStepDown: 60});
  } catch (e) {
    print('Primary already stepped down');
  }

  // Reconfigure priorities
  const config = rs.conf();
  config.members.forEach(member => {
    if (member.host.includes('${SECONDARY_REGION}')) {
      member.priority = 10;
    } else if (member.host.includes('${PRIMARY_REGION}')) {
      member.priority = 0;
    }
  });
  config.version++;

  rs.reconfig(config, {force: true});

  print('Replica set reconfigured');
"

# Step 3: Scale up application in secondary region
echo "Scaling up application in ${SECONDARY_REGION}..."
kubectl scale deployment myapp \
  --replicas=10 \
  --namespace=production \
  --context="mongodb-cluster-${SECONDARY_REGION}"

# Step 4: Wait for propagation
echo "Waiting for DNS propagation..."
sleep 60

# Step 5: Verify failover
echo "Verifying failover..."
NEW_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://api.myapp.example.com/health")

if [ "$NEW_HEALTH" = "200" ]; then
  echo "SUCCESS: Failover completed successfully"

  # Send notification
  aws sns publish \
    --topic-arn "arn:aws:sns:us-east-1:123456789012:ops-notifications" \
    --subject "Failover Completed: ${PRIMARY_REGION} -> ${SECONDARY_REGION}" \
    --message "Automated failover completed successfully at $(date)"
else
  echo "ERROR: Failover verification failed"
  exit 1
fi

echo "=========================================="
echo "Failover completed"
echo "=========================================="
```

---

## Monitoring Multi-Régions

### Prometheus Federation

```yaml
# prometheus-federation.yaml
# Federation setup for multi-region monitoring

---
# Global Prometheus (us-east-1)
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-global-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: 'global'
        region: 'us-east-1'

    # Scrape configuration for federation
    scrape_configs:
      # Federate from US region
      - job_name: 'federate-us'
        honor_labels: true
        metrics_path: '/federate'
        params:
          'match[]':
            - '{job=~"mongodb|application"}'
        static_configs:
          - targets:
              - 'prometheus-us.monitoring.svc.cluster.local:9090'

      # Federate from EU region
      - job_name: 'federate-eu'
        honor_labels: true
        metrics_path: '/federate'
        params:
          'match[]':
            - '{job=~"mongodb|application"}'
        static_configs:
          - targets:
              - 'prometheus-eu.monitoring.eu-west-1.external:9090'

      # Federate from APAC region
      - job_name: 'federate-apac'
        honor_labels: true
        metrics_path: '/federate'
        params:
          'match[]':
            - '{job=~"mongodb|application"}'
        static_configs:
          - targets:
              - 'prometheus-apac.monitoring.ap-southeast-1.external:9090'

    # Alerting rules
    rule_files:
      - '/etc/prometheus/rules/*.yml'

    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - 'alertmanager:9093'
---
# Multi-region alerting rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-multi-region-rules
  namespace: monitoring
data:
  multi-region.yml: |
    groups:
      - name: multi-region
        interval: 30s
        rules:
          # Region unavailability
          - alert: RegionUnavailable
            expr: up{job="federate"} == 0
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Region {{ $labels.region }} is unavailable"
              description: "Cannot scrape metrics from {{ $labels.region }}"

          # Cross-region replication lag
          - alert: HighReplicationLag
            expr: |
              mongodb_replset_member_optime_date{state="SECONDARY"}
              - on(replset) group_left
              mongodb_replset_member_optime_date{state="PRIMARY"}
              > 60
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High replication lag in {{ $labels.region }}"
              description: "Replication lag is {{ $value }}s"

          # Regional error rate
          - alert: HighRegionalErrorRate
            expr: |
              sum by (region) (
                rate(http_requests_total{status=~"5.."}[5m])
              )
              /
              sum by (region) (
                rate(http_requests_total[5m])
              )
              * 100 > 5
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "High error rate in {{ $labels.region }}"
              description: "Error rate is {{ $value }}%"

          # Cross-region latency
          - alert: HighCrossRegionLatency
            expr: |
              histogram_quantile(0.99,
                rate(http_request_duration_seconds_bucket{
                  source_region!~destination_region
                }[5m])
              ) > 0.5
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "High cross-region latency"
              description: "P99 latency from {{ $labels.source_region }} to {{ $labels.destination_region }} is {{ $value }}s"
```

### Grafana Multi-Region Dashboard

```json
{
  "dashboard": {
    "title": "Multi-Region Overview",
    "tags": ["multi-region", "global"],
    "panels": [
      {
        "title": "Regional Health Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"federate\"}",
            "legendFormat": "{{ region }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {
                "type": "value",
                "options": {
                  "0": { "text": "DOWN", "color": "red" },
                  "1": { "text": "UP", "color": "green" }
                }
              }
            ]
          }
        }
      },
      {
        "title": "Request Rate by Region",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (region) (rate(http_requests_total[5m]))",
            "legendFormat": "{{ region }}"
          }
        ]
      },
      {
        "title": "Error Rate by Region",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (region) (rate(http_requests_total{status=~\"5..\"}[5m])) / sum by (region) (rate(http_requests_total[5m])) * 100",
            "legendFormat": "{{ region }}"
          }
        ],
        "yaxes": [
          {
            "format": "percent",
            "label": "Error Rate"
          }
        ]
      },
      {
        "title": "Latency P99 by Region",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum by (region, le) (rate(http_request_duration_seconds_bucket[5m])))",
            "legendFormat": "{{ region }}"
          }
        ],
        "yaxes": [
          {
            "format": "s",
            "label": "Latency"
          }
        ]
      },
      {
        "title": "Cross-Region Network Traffic",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (source_region, destination_region) (rate(network_bytes_total[5m]))",
            "legendFormat": "{{ source_region }} -> {{ destination_region }}"
          }
        ]
      },
      {
        "title": "MongoDB Replication Lag",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_replset_member_optime_date{state=\"SECONDARY\"} - on(replset) group_left mongodb_replset_member_optime_date{state=\"PRIMARY\"}",
            "legendFormat": "{{ region }} - {{ member }}"
          }
        ]
      },
      {
        "title": "Active Connections by Region",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (region) (mongodb_connections{state=\"current\"})",
            "legendFormat": "{{ region }}"
          }
        ]
      },
      {
        "title": "Regional Data Distribution",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (region) (mongodb_dbstats_dataSize)"
          }
        ]
      }
    ]
  }
}
```

---

## Best Practices

### Checklist Multi-Régions

```yaml
# multi-region-best-practices.yaml
---
architecture:
  - ✅ Replica set members dans au moins 3 régions
  - ✅ Odd number of voting members (9 total)
  - ✅ Priority configuration correcte par région
  - ✅ VPC peering entre toutes les régions
  - ✅ Network latency < 100ms entre régions
  - ✅ Sufficient network bandwidth (10Gbps+)

availability:
  - ✅ Target 99.999% (five nines)
  - ✅ Automated failover configuré
  - ✅ Health checks multi-niveau
  - ✅ Circuit breakers implémentés
  - ✅ Graceful degradation
  - ✅ Chaos engineering tests réguliers

performance:
  - ✅ Read preference: nearest
  - ✅ Write concern: majority
  - ✅ Compression activée (snappy/zstd)
  - ✅ Connection pooling optimisé
  - ✅ Local caching strategy
  - ✅ CDN pour assets statiques

data_management:
  - ✅ Continuous replication monitoring
  - ✅ Replication lag < 5 seconds
  - ✅ Cross-region backup strategy
  - ✅ Point-in-time recovery tested
  - ✅ Data sovereignty compliance
  - ✅ Encryption in transit (TLS 1.3)

disaster_recovery:
  - ✅ RPO < 1 minute
  - ✅ RTO < 5 minutes
  - ✅ Automated backups every 6h
  - ✅ Backup retention 30+ days
  - ✅ Cross-region backup replication
  - ✅ DR drills quarterly
  - ✅ Runbook documentation

monitoring:
  - ✅ Prometheus federation setup
  - ✅ Multi-region dashboards
  - ✅ Alerting per region
  - ✅ SLO monitoring (99.99%)
  - ✅ Distributed tracing (Jaeger)
  - ✅ Log aggregation (ELK/Loki)
  - ✅ Cost monitoring per region

security:
  - ✅ TLS encryption entre régions
  - ✅ Network isolation (VPC peering)
  - ✅ IAM roles par région
  - ✅ Secrets rotation automatique
  - ✅ Audit logging activé
  - ✅ RBAC configuré
  - ✅ Compliance monitoring (GDPR, SOC2)

cost_optimization:
  - ✅ Reserved instances utilisées
  - ✅ Data transfer minimisé
  - ✅ Compression activée
  - ✅ Smart routing strategy
  - ✅ Storage tiering (Glacier)
  - ✅ Cost alerts configurées
  - ✅ Regular cost reviews

testing:
  - ✅ Failover tests mensuels
  - ✅ DR drills trimestriels
  - ✅ Chaos engineering
  - ✅ Load testing cross-region
  - ✅ Backup restoration tests
  - ✅ Performance benchmarks
  - ✅ Security penetration tests

documentation:
  - ✅ Architecture diagrams
  - ✅ Runbooks par scénario
  - ✅ Escalation procedures
  - ✅ Contact information
  - ✅ SLA documentation
  - ✅ Change management process
  - ✅ Post-mortem templates
```

---

## Conclusion

Les déploiements multi-régions avec MongoDB nécessitent une planification rigoureuse et une exécution précise :

**Architecture clés :**
- **Replica Set global** avec 9 membres (3 par région)
- **VPC Peering** pour connectivité cross-region
- **Global Load Balancing** avec Route53 geolocation
- **Automated Failover** avec monitoring continu

**Objectifs critiques :**
- **Availability** : 99.999% (5 nines)
- **Performance** : < 10ms local, < 100ms cross-region
- **DR** : RPO < 1min, RTO < 5min
- **Compliance** : GDPR, data sovereignty

**Stratégies essentielles :**
- Read preference: NEAREST pour performance
- Write concern: majority pour durabilité
- Compression pour réduire bandwidth
- Continuous replication monitoring
- Cross-region backups
- Automated health checks

**Infrastructure as Code :**
- Terraform multi-provider
- VPC peering automatisé
- EKS clusters multi-régions
- Route53 geo-routing
- Automated DR procedures

**Monitoring global :**
- Prometheus federation
- Multi-region dashboards
- Regional alerting
- SLO tracking
- Cost optimization

Une architecture multi-régions bien conçue garantit haute disponibilité, faible latence globale et résilience aux pannes régionales tout en maintenant la cohérence des données MongoDB.

---


⏭️ [Migration et Intégration](/19-migration-integration/README.md)
