🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.4 Kubernetes et MongoDB

## Introduction

Kubernetes est devenu la plateforme de facto pour orchestrer des applications conteneurisées en production. Déployer MongoDB sur Kubernetes présente des défis uniques liés à la nature stateful de la base de données, mais offre des avantages considérables en termes d'automatisation, de scaling et de résilience.

Cette section couvre les fondamentaux de l'orchestration MongoDB sur Kubernetes, les patterns d'architecture, et les meilleures pratiques pour des déploiements production-ready.

---

## Défis du Déploiement MongoDB sur Kubernetes

### Différences Stateless vs Stateful

```
┌───────────────────────────────────────────────────────────────┐
│              Stateless vs Stateful Applications               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Stateless (e.g., API)          Stateful (e.g., MongoDB)      │
│  ┌──────────────────┐           ┌──────────────────┐          │
│  │   Pod 1          │           │   MongoDB Pod    │          │
│  │   ┌────────┐     │           │   ┌────────┐     │          │
│  │   │  App   │     │           │   │  Data  │◀────┼─────┐    │
│  │   └────────┘     │           │   └────────┘     │     │    │
│  │   No local state │           │   Persistent     │     │    │
│  └──────────────────┘           │   Volume         │     │    │
│           │                     └──────────────────┘     │    │
│           │                              │               │    │
│           ▼                              ▼               │    │
│  • Peut être tué/recréé        • Identité stable         │    │
│  • N'importe quel nœud         • Données persistantes    │    │
│  • Scaling horizontal          • Ordre de démarrage      │    │
│  • Load balancing facile       • Network identity        │    │
│  • Pas de données locales      • Snapshots/backups       │    │
│                                                          │    │
│                                 PersistentVolume ────────┘    │
│                                 (Survit aux restarts)         │
└───────────────────────────────────────────────────────────────┘
```

### Challenges Spécifiques MongoDB

**1. Persistence et Performance**
- Les volumes Kubernetes doivent fournir des performances I/O élevées
- Les snapshots doivent être cohérents au niveau applicatif
- La latence réseau entre pods impacte la réplication

**2. Service Discovery et Networking**
- Les membres du Replica Set nécessitent des hostnames DNS stables
- Les connexions inter-pods doivent être fiables et rapides
- Les applications doivent pouvoir découvrir les membres dynamiquement

**3. Orchestration et Lifecycle**
- L'ordre de démarrage des pods est important
- Les rolling updates doivent maintenir le quorum
- Le failover doit être géré correctement

**4. Configuration et Secrets**
- Les credentials doivent être gérés de manière sécurisée
- Les keyfiles pour la réplication nécessitent une distribution coordonnée
- Les certificats TLS doivent être renouvelés automatiquement

---

## Architecture MongoDB sur Kubernetes

### Vue d'Ensemble

```
┌──────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Namespace: mongodb                       │ │
│  │                                                             │ │
│  │  ┌────────────────────────────────────────────────────────┐ │ │
│  │  │              StatefulSet: mongodb                      │ │ │
│  │  │                                                        │ │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │ │ │
│  │  │  │  Pod        │  │  Pod        │  │  Pod        │     │ │ │
│  │  │  │  mongo-0    │  │  mongo-1    │  │  mongo-2    │     │ │ │
│  │  │  │             │  │             │  │             │     │ │ │
│  │  │  │  [Primary]  │  │ [Secondary] │  │ [Secondary] │     │ │ │
│  │  │  │             │  │             │  │             │     │ │ │
│  │  │  │  Container  │  │  Container  │  │  Container  │     │ │ │
│  │  │  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │     │ │ │
│  │  │  │  │MongoDB│  │  │  │MongoDB│  │  │  │MongoDB│  │     │ │ │
│  │  │  │  └───┬───┘  │  │  └───┬───┘  │  │  └───┬───┘  │     │ │ │
│  │  │  │      │      │  │      │      │  │      │      │     │ │ │
│  │  │  │      ▼      │  │      ▼      │  │      ▼      │     │ │ │
│  │  │  │    [PVC]    │  │    [PVC]    │  │    [PVC]    │     │ │ │
│  │  │  └──────┼──────┘  └──────┼──────┘  └──────┼──────┘     │ │ │
│  │  │         │                │                │            │ │ │
│  │  │         ▼                ▼                ▼            │ │ │
│  │  │   [PV - 100Gi]    [PV - 100Gi]    [PV - 100Gi]         │ │ │
│  │  └────────────────────────────────────────────────────────┘ │ │
│  │                                                             │ │
│  │  ┌────────────────────────────────────────────────────────┐ │ │
│  │  │                   Services                             │ │ │
│  │  │                                                        │ │ │
│  │  │  ┌──────────────────┐  ┌──────────────────┐            │ │ │
│  │  │  │ Headless Service │  │  External Service│            │ │ │
│  │  │  │  mongodb-svc     │  │  mongodb-external│            │ │ │
│  │  │  │                  │  │                  │            │ │ │
│  │  │  │  • DNS per pod   │  │  • LoadBalancer  │            │ │ │
│  │  │  │  • Stable names  │  │  • External IP   │            │ │ │
│  │  │  └──────────────────┘  └──────────────────┘            │ │ │
│  │  └────────────────────────────────────────────────────────┘ │ │
│  │                                                             │ │
│  │  ┌────────────────────────────────────────────────────────┐ │ │
│  │  │                ConfigMaps & Secrets                    │ │ │
│  │  │                                                        │ │ │
│  │  │  • mongodb-config    (mongod.conf)                     │ │ │
│  │  │  • mongodb-scripts   (init scripts)                    │ │ │
│  │  │  • mongodb-secrets   (credentials, keyfile)            │ │ │
│  │  │  • mongodb-tls       (certificates)                    │ │ │
│  │  └────────────────────────────────────────────────────────┘ │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Composants Kubernetes pour MongoDB

**1. StatefulSet**
- Gère le déploiement de pods MongoDB avec identité stable
- Garantit l'ordre de création et de suppression
- Associe chaque pod à un PersistentVolumeClaim unique

**2. PersistentVolume (PV) et PersistentVolumeClaim (PVC)**
- Fournit le stockage persistant pour les données MongoDB
- Survit aux redémarrages et recreations de pods
- Peut utiliser différents backends (Local, NFS, Cloud providers)

**3. Headless Service**
- Fournit la découverte DNS pour chaque pod individuellement
- Nécessaire pour les connexions Replica Set
- Format: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

**4. ConfigMaps**
- Stocke la configuration MongoDB (mongod.conf)
- Scripts d'initialisation et de maintenance
- Configuration applicative

**5. Secrets**
- Credentials MongoDB (utilisateurs, mots de passe)
- Keyfile pour l'authentification interne du Replica Set
- Certificats TLS/SSL

**6. ServiceAccount & RBAC**
- Permissions pour l'opérateur ou les scripts d'initialisation
- Accès à l'API Kubernetes si nécessaire

---

## StatefulSets : Fondation pour MongoDB

### Anatomie d'un StatefulSet

```yaml
# mongodb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: mongodb
  labels:
    app: mongodb
spec:
  # Service responsable du DNS des pods
  serviceName: mongodb-svc

  # Nombre de replicas
  replicas: 3

  # Sélecteur pour identifier les pods
  selector:
    matchLabels:
      app: mongodb

  # Stratégie de mise à jour
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Tous les pods sont mis à jour

  # Politique de gestion des pods
  podManagementPolicy: OrderedReady  # ou Parallel

  # Template des pods
  template:
    metadata:
      labels:
        app: mongodb
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9216"

    spec:
      # Anti-affinité pour distribuer sur différents nœuds
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - mongodb
              topologyKey: kubernetes.io/hostname

      # Période de grâce pour l'arrêt propre
      terminationGracePeriodSeconds: 30

      # Init container pour préparer les données
      initContainers:
        - name: init-mongo
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              chown -R 999:999 /data/db
              chmod 700 /data/db
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db

      # Conteneur principal MongoDB
      containers:
        - name: mongodb
          image: mongo:7.0.5

          # Commande de démarrage
          command:
            - mongod
            - --replSet=rs0
            - --bind_ip_all
            - --config=/etc/mongod/mongod.conf

          # Ports exposés
          ports:
            - name: mongodb
              containerPort: 27017
              protocol: TCP

          # Variables d'environnement
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: password
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          # Volumes montés
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
            - name: mongodb-config
              mountPath: /etc/mongod
              readOnly: true
            - name: mongodb-keyfile
              mountPath: /etc/mongodb-keyfile
              readOnly: true
              subPath: keyfile
            - name: mongodb-tls
              mountPath: /etc/mongodb-tls
              readOnly: true

          # Ressources
          resources:
            requests:
              memory: "4Gi"
              cpu: "1"
            limits:
              memory: "8Gi"
              cpu: "2"

          # Probes de santé
          livenessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          # Startup probe pour première initialisation
          startupProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 30

        # Sidecar pour monitoring
        - name: mongodb-exporter
          image: percona/mongodb_exporter:0.40
          args:
            - --mongodb.uri=mongodb://localhost:27017
            - --collect-all
            - --compatible-mode
          ports:
            - name: metrics
              containerPort: 9216
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"

      # Volumes
      volumes:
        - name: mongodb-config
          configMap:
            name: mongodb-config
        - name: mongodb-keyfile
          secret:
            secretName: mongodb-keyfile
            defaultMode: 0400
        - name: mongodb-tls
          secret:
            secretName: mongodb-tls

  # Template pour les PVC (un par pod)
  volumeClaimTemplates:
    - metadata:
        name: mongodb-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

### Caractéristiques des StatefulSets

**1. Identité Stable**
```yaml
# Les pods sont nommés de manière prévisible:
mongodb-0  # Premier pod
mongodb-1  # Deuxième pod
mongodb-2  # Troisième pod

# Hostname DNS stable:
mongodb-0.mongodb-svc.mongodb.svc.cluster.local
mongodb-1.mongodb-svc.mongodb.svc.cluster.local
mongodb-2.mongodb-svc.mongodb.svc.cluster.local
```

**2. Ordre de Déploiement**
```
OrderedReady (défaut):
┌────────────────────────────────────────────────┐
│                                                │
│  Creation:  0 → Wait Ready → 1 → Wait Ready → 2│
│  Deletion:  2 → Wait Gone → 1 → Wait Gone → 0  │
│                                                │
│  • Garantit l'ordre                            │
│  • Un pod à la fois                            │
│  • Idéal pour Replica Sets                     │
└────────────────────────────────────────────────┘

Parallel:
┌────────────────────────────────────────────────┐
│                                                │
│  Creation:  0, 1, 2 → All in parallel          │
│  Deletion:  0, 1, 2 → All in parallel          │
│                                                │
│  • Plus rapide                                 │
│  • Moins de garanties                          │
│  • Attention aux race conditions               │
└────────────────────────────────────────────────┘
```

**3. Volume Persistence**
```yaml
# Chaque pod a son propre PVC:
mongodb-data-mongodb-0  → PV-1 (100Gi)
mongodb-data-mongodb-1  → PV-2 (100Gi)
mongodb-data-mongodb-2  → PV-3 (100Gi)

# Le PVC survit à la suppression du pod
# Les données sont préservées lors des recreations
```

---

## Services et Networking

### Headless Service

```yaml
# mongodb-service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
  namespace: mongodb
  labels:
    app: mongodb
spec:
  # ClusterIP: None pour service headless
  clusterIP: None

  # Publication DNS pour chaque pod
  publishNotReadyAddresses: true

  # Sélecteur des pods
  selector:
    app: mongodb

  # Ports
  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
      protocol: TCP
```

**Résolution DNS du Headless Service :**
```bash
# Requête DNS pour le service
nslookup mongodb-svc.mongodb.svc.cluster.local

# Retourne les IPs de tous les pods:
Name:    mongodb-svc.mongodb.svc.cluster.local
Address: 10.244.1.10  # mongodb-0
Address: 10.244.2.15  # mongodb-1
Address: 10.244.3.20  # mongodb-2

# Requête DNS pour un pod spécifique
nslookup mongodb-0.mongodb-svc.mongodb.svc.cluster.local

# Retourne l'IP du pod:
Name:    mongodb-0.mongodb-svc.mongodb.svc.cluster.local
Address: 10.244.1.10
```

### Service Externe pour Applications

```yaml
# mongodb-service-external.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-external
  namespace: mongodb
  labels:
    app: mongodb
spec:
  type: LoadBalancer  # ou NodePort selon le besoin

  selector:
    app: mongodb

  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
      protocol: TCP

  # Session affinity pour maintenir les connexions
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 heures
```

### Network Policies

```yaml
# mongodb-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mongodb-network-policy
  namespace: mongodb
spec:
  # Appliquer aux pods MongoDB
  podSelector:
    matchLabels:
      app: mongodb

  policyTypes:
    - Ingress
    - Egress

  # Règles d'entrée
  ingress:
    # Autoriser le trafic depuis les applications
    - from:
        - namespaceSelector:
            matchLabels:
              name: applications
        - podSelector:
            matchLabels:
              role: backend
      ports:
        - protocol: TCP
          port: 27017

    # Autoriser le trafic inter-pods MongoDB
    - from:
        - podSelector:
            matchLabels:
              app: mongodb
      ports:
        - protocol: TCP
          port: 27017

    # Autoriser Prometheus pour le monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9216

  # Règles de sortie
  egress:
    # Autoriser les requêtes DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53

    # Autoriser la communication inter-pods
    - to:
        - podSelector:
            matchLabels:
              app: mongodb
      ports:
        - protocol: TCP
          port: 27017

    # Autoriser l'accès externe (backup, monitoring externe)
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443
```

---

## Storage et Persistance

### StorageClass

```yaml
# storage-class-fast-ssd.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"

# Provisioner dépend du cloud provider
provisioner: kubernetes.io/aws-ebs  # AWS
# provisioner: kubernetes.io/gce-pd   # GCP
# provisioner: kubernetes.io/azure-disk  # Azure

parameters:
  # AWS EBS
  type: gp3
  iopsPerGB: "50"
  throughput: "250"  # MB/s
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:region:account:key/xxx"

  # GCP Persistent Disk
  # type: pd-ssd
  # replication-type: regional-pd

  # Azure Disk
  # storageaccounttype: Premium_LRS
  # kind: Managed

# Politique de récupération
reclaimPolicy: Retain  # Retain ou Delete

# Autoriser l'expansion des volumes
allowVolumeExpansion: true

# Binding immédiat ou tardif
volumeBindingMode: WaitForFirstConsumer

# Mount options
mountOptions:
  - noatime
  - nodiratime
```

### PersistentVolume Local (pour performances maximales)

```yaml
# local-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
---
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-0
  labels:
    type: local
spec:
  storageClassName: local-storage
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain

  # Référence au stockage local
  local:
    path: /mnt/disks/ssd0

  # Affinité de nœud (obligatoire pour volumes locaux)
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-1.example.com
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-1
  labels:
    type: local
spec:
  storageClassName: local-storage
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /mnt/disks/ssd0
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-2.example.com
```

### VolumeSnapshots pour Backups

```yaml
# volume-snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: mongodb-snapshot-class
driver: ebs.csi.aws.com  # Dépend du CSI driver
deletionPolicy: Retain
parameters:
  encrypted: "true"
---
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mongodb-snapshot-20240101
  namespace: mongodb
spec:
  volumeSnapshotClassName: mongodb-snapshot-class
  source:
    persistentVolumeClaimName: mongodb-data-mongodb-0
---
# restore-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-data-restored
  namespace: mongodb
spec:
  storageClassName: fast-ssd
  dataSource:
    name: mongodb-snapshot-20240101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

---

## Configuration et Secrets

### ConfigMap pour Configuration MongoDB

```yaml
# mongodb-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-config
  namespace: mongodb
data:
  mongod.conf: |
    # MongoDB Configuration
    storage:
      dbPath: /data/db
      journal:
        enabled: true
      wiredTiger:
        engineConfig:
          cacheSizeGB: 3
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
      tls:
        mode: requireTLS
        certificateKeyFile: /etc/mongodb-tls/mongodb.pem
        CAFile: /etc/mongodb-tls/ca.pem

    security:
      authorization: enabled
      keyFile: /etc/mongodb-keyfile/keyfile

    replication:
      replSetName: rs0
      oplogSizeMB: 10240

    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 100

    setParameter:
      diagnosticDataCollectionEnabled: true
      enableLocalhostAuthBypass: false
      transactionLifetimeLimitSeconds: 60

  init-replica-set.js: |
    // Script d'initialisation du Replica Set

    // Attendre que tous les membres soient disponibles
    const members = [
      { _id: 0, host: 'mongodb-0.mongodb-svc.mongodb.svc.cluster.local:27017', priority: 3 },
      { _id: 1, host: 'mongodb-1.mongodb-svc.mongodb.svc.cluster.local:27017', priority: 2 },
      { _id: 2, host: 'mongodb-2.mongodb-svc.mongodb.svc.cluster.local:27017', priority: 1 }
    ];

    // Configuration du Replica Set
    const rsConfig = {
      _id: 'rs0',
      members: members,
      settings: {
        electionTimeoutMillis: 10000,
        heartbeatIntervalMillis: 2000,
        heartbeatTimeoutSecs: 10,
        chainingAllowed: true
      }
    };

    // Initialiser le Replica Set
    try {
      rs.initiate(rsConfig);
      print('Replica Set initiated successfully');
    } catch (e) {
      if (e.code === 23) {
        print('Replica Set already initiated');
      } else {
        throw e;
      }
    }

    // Attendre l'élection du Primary
    while (true) {
      const status = rs.status();
      const primary = status.members.filter(m => m.stateStr === 'PRIMARY');
      if (primary.length > 0) {
        print('Primary elected: ' + primary[0].name);
        break;
      }
      sleep(1000);
    }
```

### Secrets Management

```yaml
# mongodb-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: mongodb
type: Opaque
stringData:
  username: admin
  password: <base64-encoded-password>

  # Connection string pour les applications
  mongodb-uri: mongodb://admin:<password>@mongodb-0.mongodb-svc,mongodb-1.mongodb-svc,mongodb-2.mongodb-svc:27017/?replicaSet=rs0&authSource=admin
---
# mongodb-keyfile-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-keyfile
  namespace: mongodb
type: Opaque
data:
  # Généré avec: openssl rand -base64 756
  keyfile: <base64-encoded-keyfile>
---
# mongodb-tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-tls
  namespace: mongodb
type: kubernetes.io/tls
data:
  # Certificat et clé privée en base64
  mongodb.pem: <base64-encoded-cert-and-key>
  ca.pem: <base64-encoded-ca-cert>
```

### Génération Automatique de Secrets

```bash
#!/bin/bash
# scripts/generate-secrets.sh

set -euo pipefail

NAMESPACE="mongodb"
SECRET_NAME="mongodb-secret"
KEYFILE_SECRET="mongodb-keyfile"

# Générer un mot de passe aléatoire
PASSWORD=$(openssl rand -base64 32)

# Créer le secret principal
kubectl create secret generic "$SECRET_NAME" \
  --from-literal=username=admin \
  --from-literal=password="$PASSWORD" \
  --namespace="$NAMESPACE" \
  --dry-run=client -o yaml | kubectl apply -f -

# Générer le keyfile pour la réplication
KEYFILE=$(openssl rand -base64 756)

kubectl create secret generic "$KEYFILE_SECRET" \
  --from-literal=keyfile="$KEYFILE" \
  --namespace="$NAMESPACE" \
  --dry-run=client -o yaml | kubectl apply -f -

echo "✅ Secrets created successfully"
echo "📝 Username: admin"
echo "🔐 Password: $PASSWORD"
```

---

## Initialisation du Replica Set

### Job d'Initialisation

```yaml
# mongodb-init-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mongodb-init
  namespace: mongodb
spec:
  # Nombre d'essais en cas d'échec
  backoffLimit: 5

  # TTL après complétion (nettoyage automatique)
  ttlSecondsAfterFinished: 600

  template:
    metadata:
      labels:
        app: mongodb-init
    spec:
      restartPolicy: OnFailure

      containers:
        - name: init
          image: mongo:7.0.5

          command:
            - bash
            - -c
            - |
              set -e

              echo "🔄 Waiting for MongoDB pods to be ready..."

              # Attendre que tous les pods MongoDB soient prêts
              for i in 0 1 2; do
                until mongosh \
                  "mongodb://mongodb-${i}.mongodb-svc.mongodb.svc.cluster.local:27017" \
                  --eval "db.adminCommand('ping')" \
                  --quiet > /dev/null 2>&1; do
                  echo "Waiting for mongodb-${i}..."
                  sleep 5
                done
                echo "✅ mongodb-${i} is ready"
              done

              echo "🚀 Initializing Replica Set..."

              # Exécuter le script d'initialisation
              mongosh \
                "mongodb://mongodb-0.mongodb-svc.mongodb.svc.cluster.local:27017" \
                /scripts/init-replica-set.js

              echo "✅ Replica Set initialized successfully"

              # Créer les utilisateurs
              echo "👤 Creating database users..."

              mongosh \
                "mongodb://mongodb-0.mongodb-svc.mongodb.svc.cluster.local:27017/admin" \
                --eval "
                  db.createUser({
                    user: '$MONGO_INITDB_ROOT_USERNAME',
                    pwd: '$MONGO_INITDB_ROOT_PASSWORD',
                    roles: [
                      { role: 'root', db: 'admin' }
                    ]
                  });
                "

              echo "✅ Database users created"

              # Vérifier l'état du Replica Set
              echo "📊 Replica Set status:"
              mongosh \
                "mongodb://$MONGO_INITDB_ROOT_USERNAME:$MONGO_INITDB_ROOT_PASSWORD@mongodb-0.mongodb-svc.mongodb.svc.cluster.local:27017/admin" \
                --eval "rs.status()"

          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: password

          volumeMounts:
            - name: init-scripts
              mountPath: /scripts
              readOnly: true

      volumes:
        - name: init-scripts
          configMap:
            name: mongodb-config
```

---

## Haute Disponibilité et Résilience

### Pod Disruption Budget

```yaml
# mongodb-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mongodb-pdb
  namespace: mongodb
spec:
  # Minimum de pods qui doivent rester disponibles
  minAvailable: 2

  # Ou: maximum de pods qui peuvent être indisponibles
  # maxUnavailable: 1

  selector:
    matchLabels:
      app: mongodb
```

**Impact du PDB :**
```
Avec minAvailable: 2 dans un Replica Set de 3 membres:
┌────────────────────────────────────────────────────┐
│                                                    │
│  Évènement      │ Pods Available │ Autorisé?       │
│  ──────────────────────────────────────────────    │
│  Node drain     │ 3 → 2          │ ✅ Oui          │
│  Upgrade        │ 2 (temporaire) │ ✅ Oui          │
│  2 nodes drain  │ 3 → 1          │ ❌ Non          │
│                                                    │
│  Le PDB protège contre:                            │
│  • Drain simultané de plusieurs nœuds              │
│  • Perte du quorum pendant la maintenance          │
│  • Disruptions involontaires                       │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Pod Anti-Affinity

```yaml
# Intégré dans le StatefulSet
spec:
  template:
    spec:
      affinity:
        # Anti-affinité stricte (required)
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - mongodb
              # Distribuer sur différents nœuds
              topologyKey: kubernetes.io/hostname

        # Anti-affinité préférée (optional)
        # podAntiAffinity:
        #   preferredDuringSchedulingIgnoredDuringExecution:
        #     - weight: 100
        #       podAffinityTerm:
        #         labelSelector:
        #           matchExpressions:
        #             - key: app
        #               operator: In
        #               values:
        #                 - mongodb
        #         topologyKey: topology.kubernetes.io/zone
```

### Topology Spread Constraints

```yaml
# Distribution sur zones de disponibilité
spec:
  template:
    spec:
      topologySpreadConstraints:
        # Distribuer équitablement sur les zones
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: mongodb

        # Distribuer équitablement sur les nœuds
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: mongodb
```

---

## Monitoring et Observabilité

### ServiceMonitor pour Prometheus

```yaml
# mongodb-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mongodb-metrics
  namespace: mongodb
  labels:
    app: mongodb
spec:
  # Sélectionner le service à monitorer
  selector:
    matchLabels:
      app: mongodb

  # Endpoints à scraper
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s

      # Relabeling
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: node

  # Namespace des pods
  namespaceSelector:
    matchNames:
      - mongodb
```

### PrometheusRule pour Alerting

```yaml
# mongodb-prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mongodb-alerts
  namespace: mongodb
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    - name: mongodb
      interval: 30s
      rules:
        # Alerte: MongoDB down
        - alert: MongoDBDown
          expr: up{job="mongodb-metrics"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "MongoDB instance down"
            description: "MongoDB instance {{ $labels.pod }} is down"

        # Alerte: Replica Set pas de Primary
        - alert: MongoDBNoPrimary
          expr: |
            sum(mongodb_mongod_replset_member_state{state="PRIMARY"}) < 1
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "MongoDB Replica Set has no PRIMARY"
            description: "Replica Set has no PRIMARY member"

        # Alerte: Replication lag élevé
        - alert: MongoDBHighReplicationLag
          expr: |
            mongodb_mongod_replset_member_replication_lag > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MongoDB high replication lag"
            description: "Replication lag is {{ $value }}s on {{ $labels.pod }}"

        # Alerte: Connexions élevées
        - alert: MongoDBHighConnections
          expr: |
            mongodb_connections{state="current"} / mongodb_connections{state="available"} > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MongoDB high connection usage"
            description: "{{ $labels.pod }} is using {{ $value | humanizePercentage }} of available connections"

        # Alerte: Utilisation disque élevée
        - alert: MongoDBHighDiskUsage
          expr: |
            (mongodb_sys_disk_used_bytes / mongodb_sys_disk_total_bytes) > 0.85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "MongoDB high disk usage"
            description: "{{ $labels.pod }} disk usage is {{ $value | humanizePercentage }}"

        # Alerte: Opérations lentes
        - alert: MongoDBSlowQueries
          expr: |
            rate(mongodb_mongod_metrics_operation_slowop_total[5m]) > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MongoDB has slow queries"
            description: "{{ $labels.pod }} has {{ $value }} slow ops per second"
```

### Grafana Dashboard ConfigMap

```yaml
# mongodb-grafana-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  mongodb-overview.json: |
    {
      "dashboard": {
        "title": "MongoDB Overview",
        "panels": [
          {
            "title": "Replica Set Status",
            "targets": [
              {
                "expr": "mongodb_mongod_replset_member_state"
              }
            ]
          },
          {
            "title": "Operations per Second",
            "targets": [
              {
                "expr": "rate(mongodb_op_counters_total[5m])"
              }
            ]
          },
          {
            "title": "Connections",
            "targets": [
              {
                "expr": "mongodb_connections"
              }
            ]
          },
          {
            "title": "Replication Lag",
            "targets": [
              {
                "expr": "mongodb_mongod_replset_member_replication_lag"
              }
            ]
          }
        ]
      }
    }
```

---

## Patterns de Déploiement

### Pattern 1 : Replica Set Multi-Zone

```yaml
# Architecture distribuée sur 3 zones
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  replicas: 3
  template:
    spec:
      # Distribuer un pod par zone
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: mongodb

      # Préférer des nœuds avec SSD
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
```

### Pattern 2 : Déploiement avec Arbiter

```yaml
# StatefulSet pour membres de données (2 replicas)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-data
spec:
  replicas: 2
  # ... configuration avec volumeClaimTemplates
---
# Deployment pour l'arbiter (sans stockage)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-arbiter
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0.5
          command:
            - mongod
            - --replSet=rs0
            - --bind_ip_all
            - --noauth
            - --port=27017
          # Pas de volume pour l'arbiter
```

### Pattern 3 : Déploiement avec Membres Hidden

```yaml
# ConfigMap avec configuration du Replica Set
data:
  init-replica-set.js: |
    const rsConfig = {
      _id: 'rs0',
      members: [
        { _id: 0, host: 'mongodb-0:27017', priority: 3 },
        { _id: 1, host: 'mongodb-1:27017', priority: 2 },
        { _id: 2, host: 'mongodb-2:27017', priority: 1 },
        // Hidden member pour analytics
        {
          _id: 3,
          host: 'mongodb-analytics:27017',
          priority: 0,
          hidden: true,
          tags: { usage: 'analytics' }
        }
      ]
    };
```

---

## Backup et Disaster Recovery

### CronJob pour Backups Automatiques

```yaml
# mongodb-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: mongodb
spec:
  # Exécution quotidienne à 2h du matin
  schedule: "0 2 * * *"

  # Historique des jobs
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: backup
              image: mongo:7.0.5

              command:
                - bash
                - -c
                - |
                  set -e

                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  BACKUP_DIR="/backup/${TIMESTAMP}"

                  echo "🔄 Starting backup: ${TIMESTAMP}"

                  # Créer le répertoire de backup
                  mkdir -p "${BACKUP_DIR}"

                  # Exécuter mongodump
                  mongodump \
                    --uri="${MONGODB_URI}" \
                    --out="${BACKUP_DIR}" \
                    --gzip \
                    --oplog

                  # Calculer la taille
                  BACKUP_SIZE=$(du -sh "${BACKUP_DIR}" | cut -f1)
                  echo "📦 Backup size: ${BACKUP_SIZE}"

                  # Upload vers S3 (si configuré)
                  if [ -n "${S3_BUCKET}" ]; then
                    echo "☁️  Uploading to S3..."

                    tar czf "/tmp/backup-${TIMESTAMP}.tar.gz" -C "/backup" "${TIMESTAMP}"

                    aws s3 cp \
                      "/tmp/backup-${TIMESTAMP}.tar.gz" \
                      "${S3_BUCKET}/mongodb-backups/" \
                      --storage-class STANDARD_IA

                    echo "✅ Backup uploaded to S3"
                  fi

                  # Cleanup des backups locaux > 7 jours
                  find /backup -type d -mtime +7 -exec rm -rf {} +

                  echo "✅ Backup completed successfully"

              env:
                - name: MONGODB_URI
                  valueFrom:
                    secretKeyRef:
                      name: mongodb-secret
                      key: mongodb-uri
                - name: S3_BUCKET
                  value: "s3://company-mongodb-backups"
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: access-key-id
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: secret-access-key

              volumeMounts:
                - name: backup-storage
                  mountPath: /backup

          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: mongodb-backup-pvc
```

### Restore Procedure

```yaml
# mongodb-restore-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mongodb-restore
  namespace: mongodb
spec:
  template:
    spec:
      restartPolicy: Never

      containers:
        - name: restore
          image: mongo:7.0.5

          command:
            - bash
            - -c
            - |
              set -e

              BACKUP_DATE="${RESTORE_BACKUP_DATE}"
              BACKUP_PATH="/backup/${BACKUP_DATE}"

              if [ ! -d "${BACKUP_PATH}" ]; then
                echo "❌ Backup not found: ${BACKUP_PATH}"
                exit 1
              fi

              echo "🔄 Restoring from backup: ${BACKUP_DATE}"

              # Arrêter les applications (optionnel)
              # kubectl scale deployment myapp --replicas=0

              # Exécuter mongorestore
              mongorestore \
                --uri="${MONGODB_URI}" \
                --dir="${BACKUP_PATH}" \
                --gzip \
                --drop \
                --oplogReplay

              echo "✅ Restore completed successfully"

          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongodb-uri
            - name: RESTORE_BACKUP_DATE
              value: "20240101_020000"  # À spécifier

          volumeMounts:
            - name: backup-storage
              mountPath: /backup
              readOnly: true

      volumes:
        - name: backup-storage
          persistentVolumeClaim:
            claimName: mongodb-backup-pvc
```

---

## Rolling Updates et Maintenance

### Stratégie de Mise à Jour

```yaml
# StatefulSet avec rolling update configuré
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # Nombre de pods pouvant être indisponibles
      partition: 0

  # Le PDB protège contre les updates trop agressifs
  # minAvailable: 2 garantit le quorum
```

**Procédure de Rolling Update :**
```bash
#!/bin/bash
# scripts/rolling-update.sh

set -euo pipefail

NAMESPACE="mongodb"
STATEFULSET="mongodb"
NEW_IMAGE="mongo:7.0.6"

echo "🔄 Starting rolling update to ${NEW_IMAGE}"

# Mettre à jour l'image
kubectl set image statefulset/${STATEFULSET} \
  mongodb=${NEW_IMAGE} \
  -n ${NAMESPACE}

# Surveiller le rollout
kubectl rollout status statefulset/${STATEFULSET} \
  -n ${NAMESPACE} \
  --timeout=10m

# Vérifier l'état du Replica Set après chaque pod
for i in 2 1 0; do
  echo "⏳ Waiting for mongodb-${i} to be updated..."

  kubectl wait pod/mongodb-${i} \
    --for=condition=Ready \
    --timeout=5m \
    -n ${NAMESPACE}

  echo "📊 Checking Replica Set status..."

  kubectl exec mongodb-0 -n ${NAMESPACE} -- \
    mongosh --eval "
      const status = rs.status();
      const healthy = status.members.filter(m => m.health === 1).length;
      const total = status.members.length;
      print('Healthy members: ' + healthy + '/' + total);

      if (healthy < 2) {
        print('ERROR: Not enough healthy members');
        quit(1);
      }
    "

  echo "✅ mongodb-${i} updated successfully"
  echo "⏱️  Waiting 30s before next pod..."
  sleep 30
done

echo "✅ Rolling update completed successfully"
```

---

## Best Practices et Recommandations

### Checklist de Production

```yaml
# production-checklist.yml
---
infrastructure:
  - ✅ StatefulSet avec au moins 3 replicas
  - ✅ StorageClass avec reclaimPolicy: Retain
  - ✅ Volumes avec IOPS suffisants (gp3, SSD)
  - ✅ PodDisruptionBudget configuré (minAvailable: 2)
  - ✅ Anti-affinity pour distribuer les pods
  - ✅ Topology constraints pour multi-zone

configuration:
  - ✅ Authentication activée
  - ✅ TLS/SSL configuré
  - ✅ Keyfile pour réplication
  - ✅ WiredTiger cache dimensionné (60% RAM)
  - ✅ Oplog size approprié (5% du disque)
  - ✅ Connection limits configurés

networking:
  - ✅ Headless service pour service discovery
  - ✅ Network policies pour isolation
  - ✅ Load balancer ou Ingress pour accès externe
  - ✅ DNS resolution testée

security:
  - ✅ Secrets pour credentials
  - ✅ RBAC configuré avec principe du moindre privilège
  - ✅ Pod Security Standards appliqués
  - ✅ Network policies restrictives
  - ✅ Audit logging activé

monitoring:
  - ✅ Prometheus ServiceMonitor
  - ✅ Alerting rules configurées
  - ✅ Grafana dashboards
  - ✅ Logging centralisé
  - ✅ Tracing distribué (optionnel)

backup:
  - ✅ Backups automatiques (CronJob)
  - ✅ Retention policy définie
  - ✅ Backup storage off-cluster
  - ✅ Restore procedure testée
  - ✅ Point-in-time recovery possible

operations:
  - ✅ Documentation à jour
  - ✅ Runbooks pour incidents
  - ✅ Procédure de rolling update
  - ✅ Disaster recovery plan
  - ✅ Monitoring et alerting testés
```

### Dimensionnement des Ressources

```yaml
# Recommandations par taille de déploiement

Small (< 100GB data, < 1000 ops/s):
  resources:
    requests:
      memory: "4Gi"
      cpu: "1"
    limits:
      memory: "8Gi"
      cpu: "2"
  storage: 100Gi

Medium (100-500GB data, 1000-5000 ops/s):
  resources:
    requests:
      memory: "16Gi"
      cpu: "4"
    limits:
      memory: "32Gi"
      cpu: "8"
  storage: 500Gi

Large (> 500GB data, > 5000 ops/s):
  resources:
    requests:
      memory: "64Gi"
      cpu: "16"
    limits:
      memory: "128Gi"
      cpu: "32"
  storage: 2Ti
```

---

## Conclusion

Le déploiement de MongoDB sur Kubernetes nécessite :

**1. Compréhension des Stateful Workloads**
- StatefulSets pour identité stable
- PersistentVolumes pour la persistance
- Service Discovery fiable

**2. Configuration Appropriée**
- Resources dimensionnées correctement
- Storage performant (SSD, IOPS élevés)
- Networking optimisé

**3. Haute Disponibilité**
- Replica Sets avec quorum
- Distribution multi-zone
- PodDisruptionBudgets

**4. Sécurité**
- Authentication et autorisation
- TLS/SSL
- Network policies
- Secrets management

**5. Observabilité**
- Monitoring avec Prometheus
- Logging centralisé
- Alerting proactif

Les sections suivantes détailleront l'utilisation des opérateurs MongoDB (Community et Enterprise) qui automatisent une grande partie de cette complexité.

---


⏭️ [MongoDB Community Kubernetes Operator](/18-devops-deploiement/04.1-community-kubernetes-operator.md)
