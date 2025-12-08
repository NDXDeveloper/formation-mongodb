🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.5 Helm Charts pour MongoDB

## Introduction

Helm est le gestionnaire de packages de facto pour Kubernetes, permettant de packager, distribuer et gérer des applications complexes comme MongoDB de manière reproductible et versionnable. Les Helm Charts définissent l'ensemble des ressources Kubernetes nécessaires au déploiement d'une application, avec un système de templating puissant et de gestion de valeurs par environnement.

```
┌────────────────────────────────────────────────────────────────────┐
│                    Helm Chart Architecture                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Helm CLI / Client                         │  │
│  │                                                              │  │
│  │  $ helm install mongodb ./mongodb-chart \                    │  │
│  │      --values production.yaml \                              │  │
│  │      --namespace mongodb \                                   │  │
│  │      --create-namespace                                      │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                        │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  Chart Structure                             │  │
│  │                                                              │  │
│  │  mongodb-chart/                                              │  │
│  │  ├── Chart.yaml          # Métadonnées du chart              │  │
│  │  ├── values.yaml         # Valeurs par défaut                │  │
│  │  ├── values-prod.yaml    # Valeurs production                │  │
│  │  ├── values-staging.yaml # Valeurs staging                   │  │
│  │  ├── charts/             # Dependencies                      │  │
│  │  ├── templates/          # Templates Kubernetes              │  │
│  │  │   ├── statefulset.yaml                                    │  │
│  │  │   ├── service.yaml                                        │  │
│  │  │   ├── configmap.yaml                                      │  │
│  │  │   ├── secrets.yaml                                        │  │
│  │  │   ├── _helpers.tpl    # Template helpers                  │  │
│  │  │   └── NOTES.txt       # Post-install notes                │  │
│  │  └── tests/              # Chart tests                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                           │                                        │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Template Rendering                              │  │
│  │                                                              │  │
│  │  values.yaml + templates/*.yaml → Kubernetes manifests       │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                        │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │           Kubernetes Resources Created                       │  │
│  │                                                              │  │
│  │  • StatefulSet (MongoDB pods)                                │  │
│  │  • Services (Headless + External)                            │  │
│  │  • ConfigMaps (Configuration)                                │  │
│  │  • Secrets (Credentials)                                     │  │
│  │  • PersistentVolumeClaims                                    │  │
│  │  • ServiceMonitor (Prometheus)                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Avantages de Helm pour MongoDB

```yaml
# helm-benefits.yaml
---
packaging:
  description: "Regrouper toutes les ressources en un seul package versionnable"
  benefits:
    - "Déploiement atomique de tous les composants"
    - "Gestion des dépendances entre ressources"
    - "Versioning sémantique du déploiement"

templating:
  description: "Templates réutilisables avec paramètres"
  benefits:
    - "Configuration par environnement (dev/staging/prod)"
    - "Réduction de la duplication de code"
    - "Validation des valeurs avec JSON Schema"

lifecycle:
  description: "Gestion du cycle de vie complet"
  benefits:
    - "Installation, upgrade, rollback facilités"
    - "Hooks pour actions pré/post déploiement"
    - "Tests intégrés pour validation"

distribution:
  description: "Distribution via repositories"
  benefits:
    - "Partage interne et externe"
    - "Versioning et release management"
    - "Chart Museum ou Harbor integration"

reusability:
  description: "Charts réutilisables et composables"
  benefits:
    - "Subchart dependencies"
    - "Override values par environnement"
    - "Community charts disponibles"
```

---

## Charts Officiels MongoDB

### MongoDB Community Chart

```bash
# Installation du chart officiel MongoDB Community
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update

# Afficher les valeurs par défaut
helm show values mongodb/community-operator

# Installation de l'opérateur
helm install community-operator mongodb/community-operator \
  --namespace mongodb-operator \
  --create-namespace

# Installation d'une instance MongoDB
helm show values mongodb/community-operator-crds

# Créer un values file personnalisé
cat > mongodb-values.yaml <<EOF
fullnameOverride: "production-mongodb"

mongodb:
  members: 3
  version: "7.0.5"

  security:
    authentication:
      modes: ["SCRAM"]
    tls:
      enabled: true

  resources:
    limits:
      cpu: "4"
      memory: "16Gi"
    requests:
      cpu: "2"
      memory: "8Gi"

  storage:
    storageClass: "fast-ssd"
    size: "500Gi"
EOF

# Installer avec les valeurs personnalisées
helm install production-mongodb mongodb/community-operator \
  --values mongodb-values.yaml \
  --namespace mongodb \
  --create-namespace
```

### MongoDB Enterprise Chart

```bash
# Ajouter le repository Enterprise
helm repo add mongodb-enterprise https://mongodb.github.io/helm-charts

# Configuration Enterprise
cat > mongodb-enterprise-values.yaml <<EOF
operator:
  name: mongodb-enterprise-operator
  namespace: mongodb-enterprise-operator
  watchNamespace: "*"

mongodb:
  name: enterprise-mongodb
  type: ReplicaSet
  members: 5
  version: "7.0.5-ent"

  opsManager:
    url: "https://ops-manager.company.com:8443"
    orgId: "5f9a8b7c6d5e4f3a2b1c0d9e"

  security:
    authentication:
      enabled: true
      modes: ["LDAP"]
    tls:
      enabled: true

  backup:
    enabled: true
    mode: "automated"

opsManagerCredentials:
  publicKey: "ABCDEFGH"
  privateKey: "1234567890abcdef"
EOF

helm install enterprise-mongodb mongodb-enterprise/mongodb-enterprise-operator \
  --values mongodb-enterprise-values.yaml \
  --namespace mongodb \
  --create-namespace
```

---

## Structure d'un Chart MongoDB Personnalisé

### Création du Chart

```bash
#!/bin/bash
# scripts/create-mongodb-chart.sh

# Créer la structure du chart
helm create mongodb-custom

# Structure générée
tree mongodb-custom/
```

### Chart.yaml - Métadonnées

```yaml
# mongodb-custom/Chart.yaml
apiVersion: v2
name: mongodb-custom
description: A production-ready MongoDB Helm chart with advanced features
type: application
version: 2.1.0  # Chart version
appVersion: "7.0.5"  # MongoDB version

# Mots-clés pour la recherche
keywords:
  - mongodb
  - database
  - nosql
  - replicaset
  - sharded

# Mainteneurs
maintainers:
  - name: DevOps Team
    email: devops@company.com
    url: https://github.com/company/mongodb-chart

# Source du chart
sources:
  - https://github.com/company/mongodb-chart
  - https://github.com/mongodb/mongodb

# URL de la documentation
home: https://docs.company.com/mongodb-chart

# Icône
icon: https://www.mongodb.com/assets/images/global/favicon.ico

# Dependencies (subcharts)
dependencies:
  # Monitoring stack
  - name: prometheus
    version: "~15.0.0"
    repository: "https://prometheus-community.github.io/helm-charts"
    condition: monitoring.prometheus.enabled
    tags:
      - monitoring

  # Grafana pour les dashboards
  - name: grafana
    version: "~6.50.0"
    repository: "https://grafana.github.io/helm-charts"
    condition: monitoring.grafana.enabled
    tags:
      - monitoring

# Annotations pour artifact hub
annotations:
  artifacthub.io/operator: "true"
  artifacthub.io/operatorCapabilities: "Full Lifecycle"
  artifacthub.io/prerelease: "false"
  artifacthub.io/recommendations: |
    - url: https://artifacthub.io/packages/helm/prometheus-community/prometheus
  artifacthub.io/links: |
    - name: Documentation
      url: https://docs.company.com/mongodb
    - name: Support
      url: https://support.company.com
```

### values.yaml - Configuration par Défaut

```yaml
# mongodb-custom/values.yaml
# Configuration par défaut pour MongoDB

# Image configuration
image:
  registry: docker.io
  repository: mongo
  tag: "7.0.5"
  pullPolicy: IfNotPresent
  pullSecrets: []

# Nom de l'installation
nameOverride: ""
fullnameOverride: ""

# MongoDB Configuration
mongodb:
  # Type de déploiement
  architecture: "replicaset"  # standalone, replicaset, sharded

  # Nombre de membres du replica set
  replicaCount: 3

  # Configuration de sécurité
  auth:
    enabled: true

    # Root credentials
    rootUser: "admin"
    rootPassword: ""  # Auto-généré si vide
    existingSecret: ""  # Utiliser un secret existant

    # Custom users
    users: []
    # - username: app-user
    #   password: ""
    #   database: app_db
    #   roles:
    #     - readWrite

    # KeyFile pour réplication interne
    keyFile: ""
    existingKeySecret: ""

  # Configuration TLS/SSL
  tls:
    enabled: false
    mode: "requireSSL"  # allowSSL, preferSSL, requireSSL

    # Certificats
    certificatesSecret: ""  # Secret contenant tls.crt, tls.key, ca.crt

    # Génération automatique de certificats (dev only)
    autoGenerate: false

  # Configuration MongoDB (mongod.conf)
  config: |
    storage:
      dbPath: /data/db
      engine: wiredTiger
      wiredTiger:
        engineConfig:
          cacheSizeGB: 4
          journalCompressor: snappy
        collectionConfig:
          blockCompressor: snappy

    net:
      port: 27017
      bindIpAll: true
      maxIncomingConnections: 65536

    replication:
      replSetName: rs0
      oplogSizeMB: 10240

    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 100

    systemLog:
      verbosity: 0
      quiet: false
      destination: file
      path: /data/logs/mongodb.log
      logAppend: true

# Resources
resources:
  limits:
    cpu: 4
    memory: 16Gi
  requests:
    cpu: 2
    memory: 8Gi

# Persistence
persistence:
  enabled: true

  # StorageClass
  storageClass: ""  # Default storage class

  # Access modes
  accessModes:
    - ReadWriteOnce

  # Size
  size: 100Gi

  # Annotations
  annotations: {}

  # Selector
  selector: {}

# Service configuration
service:
  # Service type
  type: ClusterIP

  # Port
  port: 27017

  # Headless service for StatefulSet
  headless:
    enabled: true

  # External service
  external:
    enabled: false
    type: LoadBalancer
    annotations: {}
    loadBalancerIP: ""

  # Annotations
  annotations: {}

# Pod configuration
podAnnotations: {}
podLabels: {}

podSecurityContext:
  fsGroup: 999
  fsGroupChangePolicy: "OnRootMismatch"

securityContext:
  runAsNonRoot: true
  runAsUser: 999
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: false

# Init containers
initContainers: []

# Sidecar containers
sidecars: []

# Node selection
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity
affinity:
  # Anti-affinity pour distribuer les pods
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: mongodb
          topologyKey: kubernetes.io/hostname

# Topology spread constraints
topologySpreadConstraints: []

# Priority class
priorityClassName: ""

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""

# RBAC
rbac:
  create: true

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 2

# Liveness probe
livenessProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6
  successThreshold: 1

# Readiness probe
readinessProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
  successThreshold: 1

# Startup probe
startupProbe:
  enabled: true
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 60
  successThreshold: 1

# Monitoring
monitoring:
  enabled: false

  # ServiceMonitor pour Prometheus Operator
  serviceMonitor:
    enabled: false
    interval: 30s
    scrapeTimeout: 10s
    additionalLabels: {}

  # MongoDB Exporter
  exporter:
    enabled: false
    image:
      registry: docker.io
      repository: percona/mongodb_exporter
      tag: "0.40.0"

    resources:
      limits:
        cpu: 200m
        memory: 256Mi
      requests:
        cpu: 50m
        memory: 64Mi

  # Prometheus rules
  prometheusRules:
    enabled: false
    additionalLabels: {}
    rules: []

  # Grafana dashboard
  grafanaDashboard:
    enabled: false
    configMapName: mongodb-dashboard

# Backup configuration
backup:
  enabled: false

  # CronJob schedule
  schedule: "0 2 * * *"

  # Retention
  retention:
    days: 30

  # Storage
  storage:
    type: s3  # s3, gcs, azure
    s3:
      bucket: ""
      region: ""
      endpoint: ""
      credentialsSecret: ""

  # Resources
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 1Gi

# Metrics server (optionnel)
metrics:
  enabled: false

# Tests
tests:
  enabled: true
  image:
    registry: docker.io
    repository: mongo
    tag: "7.0.5"
```

---

## Templates Avancés

### _helpers.tpl - Template Helpers

```yaml
# mongodb-custom/templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "mongodb.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mongodb.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "mongodb.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mongodb.labels" -}}
helm.sh/chart: {{ include "mongodb.chart" . }}
{{ include "mongodb.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mongodb.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mongodb.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "mongodb.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mongodb.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Return the proper MongoDB image name
*/}}
{{- define "mongodb.image" -}}
{{- $registryName := .Values.image.registry -}}
{{- $repositoryName := .Values.image.repository -}}
{{- $tag := .Values.image.tag | toString -}}
{{- printf "%s/%s:%s" $registryName $repositoryName $tag -}}
{{- end }}

{{/*
Return MongoDB root password
*/}}
{{- define "mongodb.rootPassword" -}}
{{- if .Values.mongodb.auth.existingSecret }}
{{- printf "%s" .Values.mongodb.auth.existingSecret }}
{{- else }}
{{- if .Values.mongodb.auth.rootPassword }}
{{- printf "%s" .Values.mongodb.auth.rootPassword }}
{{- else }}
{{- randAlphaNum 32 }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Return the MongoDB connection string
*/}}
{{- define "mongodb.connectionString" -}}
{{- $fullname := include "mongodb.fullname" . -}}
{{- $namespace := .Release.Namespace -}}
{{- $members := int .Values.mongodb.replicaCount -}}
{{- $hosts := list -}}
{{- range $i := until $members -}}
{{- $hosts = append $hosts (printf "%s-%d.%s-headless.%s.svc.cluster.local:27017" $fullname $i $fullname $namespace) -}}
{{- end -}}
mongodb://{{ join "," $hosts }}/{{ if .Values.mongodb.auth.enabled }}?authSource=admin&replicaSet=rs0{{ else }}?replicaSet=rs0{{ end }}
{{- end }}

{{/*
Compile all warnings into a single message
*/}}
{{- define "mongodb.validateValues" -}}
{{- $messages := list -}}
{{- if and .Values.mongodb.auth.enabled (not .Values.mongodb.auth.rootPassword) (not .Values.mongodb.auth.existingSecret) -}}
{{- $messages = append $messages "WARNING: No root password specified. One will be auto-generated." -}}
{{- end -}}
{{- if and .Values.mongodb.tls.enabled (not .Values.mongodb.tls.certificatesSecret) (not .Values.mongodb.tls.autoGenerate) -}}
{{- $messages = append $messages "ERROR: TLS enabled but no certificates provided" -}}
{{- end -}}
{{- if lt (int .Values.mongodb.replicaCount) 3 -}}
{{- $messages = append $messages "WARNING: Replica count is less than 3. This is not recommended for production." -}}
{{- end -}}
{{- if $messages -}}
{{- printf "\nVALIDATION:\n%s" (join "\n" $messages) | fail -}}
{{- end -}}
{{- end -}}
```

### statefulset.yaml - StatefulSet Template

```yaml
# mongodb-custom/templates/statefulset.yaml
{{- $_ := include "mongodb.validateValues" . -}}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "mongodb.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
spec:
  serviceName: {{ include "mongodb.fullname" . }}-headless
  replicas: {{ .Values.mongodb.replicaCount }}

  podManagementPolicy: OrderedReady

  updateStrategy:
    type: RollingUpdate

  selector:
    matchLabels:
      {{- include "mongodb.selectorLabels" . | nindent 6 }}

  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "mongodb.selectorLabels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      serviceAccountName: {{ include "mongodb.serviceAccountName" . }}

      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      terminationGracePeriodSeconds: 30

      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- if .Values.priorityClassName }}
      priorityClassName: {{ .Values.priorityClassName }}
      {{- end }}

      initContainers:
        # Préparer les permissions
        - name: init-permissions
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              chown -R 999:999 /data/db /data/logs
              chmod 750 /data/db /data/logs
          volumeMounts:
            - name: data
              mountPath: /data/db
            - name: logs
              mountPath: /data/logs
          securityContext:
            runAsUser: 0

        {{- with .Values.initContainers }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

      containers:
        # MongoDB
        - name: mongodb
          image: {{ include "mongodb.image" . }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}

          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          command:
            - mongod

          args:
            - --config=/etc/mongodb/mongod.conf
            {{- if .Values.mongodb.auth.enabled }}
            - --auth
            - --keyFile=/etc/mongodb/keyfile/mongodb-keyfile
            {{- end }}
            {{- if eq .Values.mongodb.architecture "replicaset" }}
            - --replSet=rs0
            {{- end }}
            {{- if .Values.mongodb.tls.enabled }}
            - --tlsMode={{ .Values.mongodb.tls.mode }}
            - --tlsCertificateKeyFile=/etc/mongodb/tls/tls.pem
            - --tlsCAFile=/etc/mongodb/tls/ca.crt
            {{- end }}

          ports:
            - name: mongodb
              containerPort: 27017
              protocol: TCP

          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name

            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

            {{- if .Values.mongodb.auth.enabled }}
            - name: MONGO_INITDB_ROOT_USERNAME
              value: {{ .Values.mongodb.auth.rootUser }}

            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "mongodb.fullname" . }}
                  key: mongodb-root-password
            {{- end }}

          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          volumeMounts:
            - name: data
              mountPath: /data/db

            - name: logs
              mountPath: /data/logs

            - name: config
              mountPath: /etc/mongodb
              readOnly: true

            {{- if .Values.mongodb.auth.enabled }}
            - name: keyfile
              mountPath: /etc/mongodb/keyfile
              readOnly: true
            {{- end }}

            {{- if .Values.mongodb.tls.enabled }}
            - name: tls
              mountPath: /etc/mongodb/tls
              readOnly: true
            {{- end }}

          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
            successThreshold: {{ .Values.livenessProbe.successThreshold }}
          {{- end }}

          {{- if .Values.readinessProbe.enabled }}
          readinessProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
            successThreshold: {{ .Values.readinessProbe.successThreshold }}
          {{- end }}

          {{- if .Values.startupProbe.enabled }}
          startupProbe:
            exec:
              command:
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: {{ .Values.startupProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.startupProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.startupProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.startupProbe.failureThreshold }}
            successThreshold: {{ .Values.startupProbe.successThreshold }}
          {{- end }}

        {{- if .Values.monitoring.exporter.enabled }}
        # MongoDB Exporter
        - name: mongodb-exporter
          image: {{ .Values.monitoring.exporter.image.registry }}/{{ .Values.monitoring.exporter.image.repository }}:{{ .Values.monitoring.exporter.image.tag }}

          args:
            - --mongodb.uri=mongodb://$(MONGODB_ROOT_USER):$(MONGODB_ROOT_PASSWORD)@localhost:27017/admin
            - --collect-all
            - --compatible-mode

          ports:
            - name: metrics
              containerPort: 9216
              protocol: TCP

          env:
            - name: MONGODB_ROOT_USER
              value: {{ .Values.mongodb.auth.rootUser }}

            - name: MONGODB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "mongodb.fullname" . }}
                  key: mongodb-root-password

          resources:
            {{- toYaml .Values.monitoring.exporter.resources | nindent 12 }}
        {{- end }}

        {{- with .Values.sidecars }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

      volumes:
        - name: config
          configMap:
            name: {{ include "mongodb.fullname" . }}-config

        - name: logs
          emptyDir: {}

        {{- if .Values.mongodb.auth.enabled }}
        - name: keyfile
          secret:
            secretName: {{ include "mongodb.fullname" . }}-keyfile
            defaultMode: 0400
        {{- end }}

        {{- if .Values.mongodb.tls.enabled }}
        - name: tls
          secret:
            secretName: {{ .Values.mongodb.tls.certificatesSecret }}
        {{- end }}

  volumeClaimTemplates:
    - metadata:
        name: data
        {{- with .Values.persistence.annotations }}
        annotations:
          {{- toYaml . | nindent 10 }}
        {{- end }}
      spec:
        accessModes:
          {{- range .Values.persistence.accessModes }}
          - {{ . | quote }}
          {{- end }}

        {{- if .Values.persistence.storageClass }}
        storageClassName: {{ .Values.persistence.storageClass }}
        {{- end }}

        resources:
          requests:
            storage: {{ .Values.persistence.size }}

        {{- with .Values.persistence.selector }}
        selector:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

### service.yaml - Services Template

```yaml
# mongodb-custom/templates/service.yaml
{{- if .Values.service.headless.enabled }}
---
# Headless Service pour StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mongodb.fullname" . }}-headless
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
    service-type: headless
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: ClusterIP
  clusterIP: None
  publishNotReadyAddresses: true

  ports:
    - name: mongodb
      port: {{ .Values.service.port }}
      targetPort: mongodb
      protocol: TCP

  selector:
    {{- include "mongodb.selectorLabels" . | nindent 4 }}
{{- end }}

---
# Service standard
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mongodb.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
    service-type: standard
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}

  {{- if and (eq .Values.service.type "LoadBalancer") .Values.service.loadBalancerIP }}
  loadBalancerIP: {{ .Values.service.loadBalancerIP }}
  {{- end }}

  ports:
    - name: mongodb
      port: {{ .Values.service.port }}
      targetPort: mongodb
      protocol: TCP

  selector:
    {{- include "mongodb.selectorLabels" . | nindent 4 }}

{{- if .Values.service.external.enabled }}
---
# External Service (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mongodb.fullname" . }}-external
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
    service-type: external
  {{- with .Values.service.external.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.external.type }}

  {{- if .Values.service.external.loadBalancerIP }}
  loadBalancerIP: {{ .Values.service.external.loadBalancerIP }}
  {{- end }}

  ports:
    - name: mongodb
      port: {{ .Values.service.port }}
      targetPort: mongodb
      protocol: TCP

  selector:
    {{- include "mongodb.selectorLabels" . | nindent 4 }}
{{- end }}
```

---

## Values Files par Environnement

### values-dev.yaml - Développement

```yaml
# mongodb-custom/values-dev.yaml
# Configuration pour environnement de développement

image:
  pullPolicy: Always

mongodb:
  replicaCount: 1  # Single node pour dev

  auth:
    enabled: true
    rootPassword: "dev-password-123"  # Mot de passe simple pour dev

  tls:
    enabled: false  # Pas de TLS en dev

  config: |
    storage:
      engine: wiredTiger
      wiredTiger:
        engineConfig:
          cacheSizeGB: 0.5  # Cache réduit pour dev

    net:
      port: 27017
      bindIpAll: true

    systemLog:
      verbosity: 1  # Plus de logs en dev
      quiet: false

resources:
  limits:
    cpu: 500m
    memory: 1Gi
  requests:
    cpu: 250m
    memory: 512Mi

persistence:
  enabled: true
  storageClass: "standard"
  size: 10Gi

# Pas d'anti-affinity en dev
affinity: {}

# Monitoring désactivé en dev
monitoring:
  enabled: false

# Backup désactivé en dev
backup:
  enabled: false

# PDB désactivé en dev
podDisruptionBudget:
  enabled: false
```

### values-staging.yaml - Staging

```yaml
# mongodb-custom/values-staging.yaml
# Configuration pour environnement de staging

image:
  pullPolicy: IfNotPresent

mongodb:
  replicaCount: 3

  auth:
    enabled: true
    # Utiliser un secret existant
    existingSecret: "mongodb-staging-credentials"

  tls:
    enabled: true
    mode: "requireSSL"
    certificatesSecret: "mongodb-staging-tls"

  config: |
    storage:
      engine: wiredTiger
      wiredTiger:
        engineConfig:
          cacheSizeGB: 2
          journalCompressor: snappy

    net:
      port: 27017
      bindIpAll: true
      maxIncomingConnections: 10000

    replication:
      replSetName: rs0
      oplogSizeMB: 5120

    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 100

resources:
  limits:
    cpu: 2
    memory: 8Gi
  requests:
    cpu: 1
    memory: 4Gi

persistence:
  enabled: true
  storageClass: "fast-ssd"
  size: 100Gi

# Anti-affinity préférée
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: mongodb
          topologyKey: kubernetes.io/hostname

# Monitoring activé
monitoring:
  enabled: true
  exporter:
    enabled: true
  serviceMonitor:
    enabled: true

# Backup activé avec retention courte
backup:
  enabled: true
  schedule: "0 3 * * *"  # 3h du matin
  retention:
    days: 7

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

### values-prod.yaml - Production

```yaml
# mongodb-custom/values-prod.yaml
# Configuration pour environnement de production

image:
  pullPolicy: IfNotPresent
  # Tag spécifique pour production
  tag: "7.0.5"

mongodb:
  replicaCount: 5  # 5 membres pour haute disponibilité

  auth:
    enabled: true
    existingSecret: "mongodb-prod-credentials"

    # Utilisateurs custom
    users:
      - username: app-user
        password: ""
        database: production_db
        existingSecret: "app-user-credentials"
        roles:
          - readWrite

      - username: backup-user
        password: ""
        database: admin
        existingSecret: "backup-user-credentials"
        roles:
          - backup
          - restore

      - username: monitoring-user
        password: ""
        database: admin
        existingSecret: "monitoring-user-credentials"
        roles:
          - clusterMonitor

  tls:
    enabled: true
    mode: "requireSSL"
    certificatesSecret: "mongodb-prod-tls"

  config: |
    storage:
      engine: wiredTiger
      wiredTiger:
        engineConfig:
          cacheSizeGB: 12  # 60% de la RAM
          journalCompressor: snappy
        collectionConfig:
          blockCompressor: snappy
        indexConfig:
          prefixCompression: true

    net:
      port: 27017
      bindIpAll: true
      maxIncomingConnections: 65536
      compression:
        compressors: "snappy,zstd,zlib"

    replication:
      replSetName: rs0
      oplogSizeMB: 20480  # 20GB oplog

    operationProfiling:
      mode: slowOp
      slowOpThresholdMs: 50

    systemLog:
      verbosity: 0
      quiet: false
      destination: file
      path: /data/logs/mongodb.log
      logAppend: true

    setParameter:
      diagnosticDataCollectionEnabled: true
      transactionLifetimeLimitSeconds: 60

resources:
  limits:
    cpu: 8
    memory: 32Gi
  requests:
    cpu: 4
    memory: 16Gi

persistence:
  enabled: true
  storageClass: "premium-ssd"  # Storage le plus performant
  size: 1Ti

  annotations:
    volume.beta.kubernetes.io/storage-class: "premium-ssd"

service:
  type: ClusterIP

  headless:
    enabled: true

  external:
    enabled: false

# Node selector pour dédier des nœuds à MongoDB
nodeSelector:
  workload: database
  disktype: ssd

# Tolerations pour les nœuds dédiés
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"

# Anti-affinity stricte
affinity:
  # Distribuer les pods sur différents nœuds
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: mongodb
        topologyKey: kubernetes.io/hostname

  # Préférer des nœuds dans différentes zones
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - eu-west-3a
                - eu-west-3b
                - eu-west-3c

# Topology spread multi-zone
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: mongodb

# Priority class haute pour production
priorityClassName: "high-priority"

# Monitoring complet
monitoring:
  enabled: true

  exporter:
    enabled: true
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 200m
        memory: 256Mi

  serviceMonitor:
    enabled: true
    interval: 30s
    scrapeTimeout: 10s
    additionalLabels:
      prometheus: main

  prometheusRules:
    enabled: true
    additionalLabels:
      prometheus: main
    rules:
      - alert: MongoDBDown
        expr: up{job="mongodb"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB instance is down"

      - alert: MongoDBHighReplicationLag
        expr: mongodb_replset_member_replication_lag > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High replication lag detected"

      - alert: MongoDBHighConnections
        expr: (mongodb_connections{state="current"} / mongodb_connections{state="available"}) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High connection usage"

  grafanaDashboard:
    enabled: true

# Backup automatisé
backup:
  enabled: true
  schedule: "0 2 * * *"  # 2h du matin quotidiennement

  retention:
    days: 30

  storage:
    type: s3
    s3:
      bucket: "company-mongodb-backups"
      region: "eu-west-3"
      endpoint: "https://s3.eu-west-3.amazonaws.com"
      credentialsSecret: "aws-backup-credentials"

  resources:
    limits:
      cpu: 2
      memory: 4Gi
    requests:
      cpu: 1
      memory: 2Gi

# PDB strict
podDisruptionBudget:
  enabled: true
  minAvailable: 3  # Maintenir le quorum

# Probes ajustées pour production
livenessProbe:
  enabled: true
  initialDelaySeconds: 60
  periodSeconds: 20
  timeoutSeconds: 10
  failureThreshold: 3

readinessProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

startupProbe:
  enabled: true
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 120  # 20 minutes max pour démarrer
```

---

## Helm Hooks et Lifecycle

### Pre-Install Hook - Initialisation

```yaml
# mongodb-custom/templates/hooks/pre-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mongodb.fullname" . }}-pre-install
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 3

  template:
    spec:
      restartPolicy: Never

      containers:
        - name: pre-install
          image: busybox:1.35

          command:
            - sh
            - -c
            - |
              echo "🚀 Pre-install hook started"

              # Vérifier les prérequis
              echo "Checking prerequisites..."

              # Vérifier la StorageClass
              if [ -n "{{ .Values.persistence.storageClass }}" ]; then
                echo "✅ StorageClass configured: {{ .Values.persistence.storageClass }}"
              else
                echo "⚠️  No StorageClass specified, using default"
              fi

              # Vérifier les secrets requis
              {{- if .Values.mongodb.auth.existingSecret }}
              echo "✅ Using existing secret: {{ .Values.mongodb.auth.existingSecret }}"
              {{- else }}
              echo "ℹ️  Credentials will be auto-generated"
              {{- end }}

              echo "✅ Pre-install checks completed"
```

### Post-Install Hook - Initialisation Replica Set

```yaml
# mongodb-custom/templates/hooks/post-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mongodb.fullname" . }}-post-install
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 5

  template:
    spec:
      restartPolicy: Never

      containers:
        - name: post-install
          image: {{ include "mongodb.image" . }}

          command:
            - sh
            - -c
            - |
              set -e

              echo "🔄 Post-install: Initializing replica set..."

              # Attendre que tous les pods soient prêts
              REPLICAS={{ .Values.mongodb.replicaCount }}
              echo "Waiting for $REPLICAS pods to be ready..."

              for i in $(seq 0 $((REPLICAS-1))); do
                POD_NAME="{{ include "mongodb.fullname" . }}-${i}"

                until mongosh --host "$POD_NAME.{{ include "mongodb.fullname" . }}-headless" \
                  --eval "db.adminCommand('ping')" > /dev/null 2>&1; do
                  echo "Waiting for $POD_NAME..."
                  sleep 5
                done

                echo "✅ $POD_NAME is ready"
              done

              {{- if eq .Values.mongodb.architecture "replicaset" }}
              # Initialiser le replica set
              echo "Initializing replica set..."

              mongosh --host "{{ include "mongodb.fullname" . }}-0.{{ include "mongodb.fullname" . }}-headless" \
                {{- if .Values.mongodb.auth.enabled }}
                -u "$MONGODB_ROOT_USER" -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase admin \
                {{- end }}
                --eval "
                  rs.status().ok || rs.initiate({
                    _id: 'rs0',
                    members: [
                      {{- range $i := until (int .Values.mongodb.replicaCount) }}
                      {
                        _id: {{ $i }},
                        host: '{{ include "mongodb.fullname" $ }}-{{ $i }}.{{ include "mongodb.fullname" $ }}-headless.{{ $.Release.Namespace }}.svc.cluster.local:27017',
                        priority: {{ sub (int $.Values.mongodb.replicaCount) $i }}
                      }{{ if ne $i (sub (int $.Values.mongodb.replicaCount) 1) }},{{ end }}
                      {{- end }}
                    ]
                  })
                "

              # Attendre l'élection du primary
              echo "Waiting for primary election..."

              for i in {1..60}; do
                STATUS=$(mongosh --host "{{ include "mongodb.fullname" . }}-0.{{ include "mongodb.fullname" . }}-headless" \
                  {{- if .Values.mongodb.auth.enabled }}
                  -u "$MONGODB_ROOT_USER" -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase admin \
                  {{- end }}
                  --quiet --eval "rs.status().myState")

                if [ "$STATUS" == "1" ]; then
                  echo "✅ Primary elected"
                  break
                fi

                echo "Waiting for primary election... ($i/60)"
                sleep 2
              done

              # Afficher le statut final
              mongosh --host "{{ include "mongodb.fullname" . }}-0.{{ include "mongodb.fullname" . }}-headless" \
                {{- if .Values.mongodb.auth.enabled }}
                -u "$MONGODB_ROOT_USER" -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase admin \
                {{- end }}
                --eval "rs.status()"
              {{- end }}

              echo "✅ Post-install completed successfully"

          env:
            {{- if .Values.mongodb.auth.enabled }}
            - name: MONGODB_ROOT_USER
              value: {{ .Values.mongodb.auth.rootUser }}

            - name: MONGODB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "mongodb.fullname" . }}
                  key: mongodb-root-password
            {{- end }}
```

### Pre-Upgrade Hook - Backup

```yaml
# mongodb-custom/templates/hooks/pre-upgrade-job.yaml
{{- if .Values.backup.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mongodb.fullname" . }}-pre-upgrade-backup
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  ttlSecondsAfterFinished: 3600
  backoffLimit: 2

  template:
    spec:
      restartPolicy: Never

      containers:
        - name: pre-upgrade-backup
          image: {{ include "mongodb.image" . }}

          command:
            - sh
            - -c
            - |
              set -e

              echo "💾 Pre-upgrade: Creating backup..."

              BACKUP_NAME="pre-upgrade-$(date +%Y%m%d-%H%M%S)"
              BACKUP_DIR="/backup/$BACKUP_NAME"

              # Créer le backup
              mongodump \
                --uri="mongodb://{{ include "mongodb.fullname" . }}-0.{{ include "mongodb.fullname" . }}-headless:27017" \
                {{- if .Values.mongodb.auth.enabled }}
                --username="$MONGODB_ROOT_USER" \
                --password="$MONGODB_ROOT_PASSWORD" \
                --authenticationDatabase=admin \
                {{- end }}
                --out="$BACKUP_DIR" \
                --gzip \
                --oplog

              # Upload vers S3 si configuré
              {{- if eq .Values.backup.storage.type "s3" }}
              echo "Uploading to S3..."

              tar czf "${BACKUP_NAME}.tar.gz" -C /backup "$BACKUP_NAME"

              aws s3 cp "${BACKUP_NAME}.tar.gz" \
                "s3://{{ .Values.backup.storage.s3.bucket }}/pre-upgrade/" \
                --metadata "upgrade-timestamp=$(date -u +%Y%m%dT%H%M%SZ),chart-version={{ .Chart.Version }}"

              echo "✅ Backup uploaded to S3"
              {{- end }}

              echo "✅ Pre-upgrade backup completed"

          env:
            {{- if .Values.mongodb.auth.enabled }}
            - name: MONGODB_ROOT_USER
              value: {{ .Values.mongodb.auth.rootUser }}

            - name: MONGODB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "mongodb.fullname" . }}
                  key: mongodb-root-password
            {{- end }}

            {{- if eq .Values.backup.storage.type "s3" }}
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.backup.storage.s3.credentialsSecret }}
                  key: access-key-id

            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.backup.storage.s3.credentialsSecret }}
                  key: secret-access-key

            - name: AWS_DEFAULT_REGION
              value: {{ .Values.backup.storage.s3.region }}
            {{- end }}

          volumeMounts:
            - name: backup
              mountPath: /backup

      volumes:
        - name: backup
          emptyDir:
            sizeLimit: 100Gi
{{- end }}
```

---

## Chart Testing

### tests/test-connection.yaml

```yaml
# mongodb-custom/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "mongodb.fullname" . }}-test-connection
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mongodb.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never

  containers:
    - name: test-connection
      image: {{ include "mongodb.image" . }}

      command:
        - sh
        - -c
        - |
          set -e

          echo "🧪 Testing MongoDB connection..."

          # Test de connexion basique
          mongosh --host "{{ include "mongodb.fullname" . }}" \
            {{- if .Values.mongodb.auth.enabled }}
            -u "$MONGODB_ROOT_USER" -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase admin \
            {{- end }}
            --eval "db.adminCommand('ping')"

          echo "✅ Basic connection test passed"

          {{- if eq .Values.mongodb.architecture "replicaset" }}
          # Test du replica set
          echo "Testing replica set status..."

          mongosh --host "{{ include "mongodb.fullname" . }}" \
            {{- if .Values.mongodb.auth.enabled }}
            -u "$MONGODB_ROOT_USER" -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase admin \
            {{- end }}
            --eval "
              const status = rs.status();
              if (status.ok !== 1) {
                throw new Error('Replica set status not OK');
              }

              const primary = status.members.find(m => m.state === 1);
              if (!primary) {
                throw new Error('No primary found');
              }

              print('✅ Primary: ' + primary.name);
              print('✅ Members: ' + status.members.length);
            "

          echo "✅ Replica set test passed"
          {{- end }}

          # Test de lecture/écriture
          echo "Testing read/write operations..."

          mongosh --host "{{ include "mongodb.fullname" . }}" \
            {{- if .Values.mongodb.auth.enabled }}
            -u "$MONGODB_ROOT_USER" -p "$MONGODB_ROOT_PASSWORD" --authenticationDatabase admin \
            {{- end }}
            --eval "
              use test;

              // Write test
              db.helm_test.insertOne({
                timestamp: new Date(),
                test: 'helm-chart-test',
                version: '{{ .Chart.Version }}'
              });

              // Read test
              const doc = db.helm_test.findOne({test: 'helm-chart-test'});
              if (!doc) {
                throw new Error('Document not found');
              }

              // Cleanup
              db.helm_test.deleteMany({test: 'helm-chart-test'});

              print('✅ Read/write test passed');
            "

          echo "✅ All tests passed!"

      env:
        {{- if .Values.mongodb.auth.enabled }}
        - name: MONGODB_ROOT_USER
          value: {{ .Values.mongodb.auth.rootUser }}

        - name: MONGODB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "mongodb.fullname" . }}
              key: mongodb-root-password
        {{- end }}
```

---

## NOTES.txt - Instructions Post-Installation

```txt
# mongodb-custom/templates/NOTES.txt
{{- $fullname := include "mongodb.fullname" . -}}
{{- $namespace := .Release.Namespace -}}

███╗   ███╗ ██████╗ ███╗   ██╗ ██████╗  ██████╗ ██████╗ ██████╗
████╗ ████║██╔═══██╗████╗  ██║██╔════╝ ██╔═══██╗██╔══██╗██╔══██╗
██╔████╔██║██║   ██║██╔██╗ ██║██║  ███╗██║   ██║██║  ██║██████╔╝
██║╚██╔╝██║██║   ██║██║╚██╗██║██║   ██║██║   ██║██║  ██║██╔══██╗
██║ ╚═╝ ██║╚██████╔╝██║ ╚████║╚██████╔╝╚██████╔╝██████╔╝██████╔╝
╚═╝     ╚═╝ ╚═════╝ ╚═╝  ╚═══╝ ╚═════╝  ╚═════╝ ╚═════╝ ╚═════╝

MongoDB has been deployed successfully!

Chart: {{ .Chart.Name }}-{{ .Chart.Version }}
Release: {{ .Release.Name }}
Namespace: {{ $namespace }}

📊 Deployment Information:
==========================

Architecture: {{ .Values.mongodb.architecture }}
{{- if eq .Values.mongodb.architecture "replicaset" }}
Replica Set Members: {{ .Values.mongodb.replicaCount }}
{{- end }}
MongoDB Version: {{ .Values.image.tag }}

🔐 Security:
===========

{{- if .Values.mongodb.auth.enabled }}
✅ Authentication: ENABLED

Root Username: {{ .Values.mongodb.auth.rootUser }}
{{- if .Values.mongodb.auth.existingSecret }}
Root Password: (stored in secret: {{ .Values.mongodb.auth.existingSecret }})
{{- else }}
Root Password: (generated, retrieve with command below)

  kubectl get secret {{ $fullname }} \
    -n {{ $namespace }} \
    -o jsonpath='{.data.mongodb-root-password}' | base64 -d
{{- end }}
{{- else }}
⚠️  Authentication: DISABLED (not recommended for production)
{{- end }}

{{- if .Values.mongodb.tls.enabled }}
✅ TLS/SSL: ENABLED ({{ .Values.mongodb.tls.mode }})
{{- else }}
⚠️  TLS/SSL: DISABLED
{{- end }}

🌐 Connection Information:
=========================

{{- if eq .Values.mongodb.architecture "replicaset" }}
MongoDB Connection String:
  {{ include "mongodb.connectionString" . }}

Connect via mongosh:
  mongosh "{{ include "mongodb.connectionString" . }}" \
    {{- if .Values.mongodb.auth.enabled }}
    -u {{ .Values.mongodb.auth.rootUser }} \
    -p <password>
    {{- end }}

From within the cluster:
  kubectl run mongodb-client --rm -it --restart=Never \
    --namespace {{ $namespace }} \
    --image {{ include "mongodb.image" . }} \
    -- mongosh "{{ include "mongodb.connectionString" . }}" \
    {{- if .Values.mongodb.auth.enabled }}
    -u {{ .Values.mongodb.auth.rootUser }} -p <password>
    {{- end }}
{{- else }}
MongoDB Host: {{ $fullname }}.{{ $namespace }}.svc.cluster.local
MongoDB Port: {{ .Values.service.port }}

Connect via mongosh:
  mongosh --host {{ $fullname }}.{{ $namespace }}.svc.cluster.local:{{ .Values.service.port }} \
    {{- if .Values.mongodb.auth.enabled }}
    -u {{ .Values.mongodb.auth.rootUser }} -p <password> --authenticationDatabase admin
    {{- end }}
{{- end }}

📦 Resources:
============

Pods:
{{- range $i := until (int .Values.mongodb.replicaCount) }}
  • {{ $fullname }}-{{ $i }}
{{- end }}

Services:
  • {{ $fullname }} ({{ .Values.service.type }})
  • {{ $fullname }}-headless (ClusterIP - Headless)
{{- if .Values.service.external.enabled }}
  • {{ $fullname }}-external ({{ .Values.service.external.type }})
{{- end }}

PersistentVolumeClaims:
{{- range $i := until (int .Values.mongodb.replicaCount) }}
  • data-{{ $fullname }}-{{ $i }} ({{ .Values.persistence.size }})
{{- end }}

🔍 Monitoring:
=============

{{- if .Values.monitoring.enabled }}
✅ Monitoring: ENABLED

{{- if .Values.monitoring.exporter.enabled }}
MongoDB Exporter metrics available at:
  http://{{ $fullname }}-<pod-number>:9216/metrics
{{- end }}

{{- if .Values.monitoring.serviceMonitor.enabled }}
✅ ServiceMonitor created for Prometheus Operator
{{- end }}

{{- if .Values.monitoring.grafanaDashboard.enabled }}
✅ Grafana Dashboard: {{ .Values.monitoring.grafanaDashboard.configMapName }}
{{- end }}
{{- else }}
⚠️  Monitoring: DISABLED
{{- end }}

💾 Backup:
=========

{{- if .Values.backup.enabled }}
✅ Backup: ENABLED

Schedule: {{ .Values.backup.schedule }}
Retention: {{ .Values.backup.retention.days }} days
Storage: {{ .Values.backup.storage.type }}
{{- if eq .Values.backup.storage.type "s3" }}
S3 Bucket: {{ .Values.backup.storage.s3.bucket }}
{{- end }}
{{- else }}
⚠️  Backup: DISABLED
{{- end }}

📝 Useful Commands:
==================

# Check pod status
kubectl get pods -n {{ $namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

# View logs
kubectl logs -n {{ $namespace }} {{ $fullname }}-0 -f

# Execute commands in pod
kubectl exec -n {{ $namespace }} {{ $fullname }}-0 -it -- mongosh

{{- if eq .Values.mongodb.architecture "replicaset" }}
# Check replica set status
kubectl exec -n {{ $namespace }} {{ $fullname }}-0 -- \
  mongosh {{- if .Values.mongodb.auth.enabled }} -u {{ .Values.mongodb.auth.rootUser }} -p <password> --authenticationDatabase admin{{- end }} \
  --eval "rs.status()"
{{- end }}

# Port forward to access locally
kubectl port-forward -n {{ $namespace }} svc/{{ $fullname }} 27017:27017

📚 Documentation:
================

Chart Repository: {{ .Chart.Home }}
Chart Version: {{ .Chart.Version }}
App Version: {{ .Chart.AppVersion }}

{{- if .Chart.Sources }}
Sources:
{{- range .Chart.Sources }}
  • {{ . }}
{{- end }}
{{- end }}

For more information, visit: https://docs.company.com/mongodb-chart

⚠️  Important Notes:
===================

{{- if not .Values.mongodb.auth.enabled }}
⚠️  Authentication is DISABLED. This is not recommended for production!
{{- end }}

{{- if not .Values.mongodb.tls.enabled }}
⚠️  TLS/SSL is DISABLED. Enable TLS for production environments.
{{- end }}

{{- if lt (int .Values.mongodb.replicaCount) 3 }}
⚠️  Replica count is less than 3. Consider increasing for high availability.
{{- end }}

{{- if not .Values.persistence.enabled }}
⚠️  Persistence is DISABLED. Data will be lost when pods are deleted!
{{- end }}

{{- if not .Values.backup.enabled }}
⚠️  Backups are DISABLED. Enable backups for production environments.
{{- end }}

Happy MongoDB-ing! 🎉
```

---

## Installation et Gestion du Chart

### Installation

```bash
#!/bin/bash
# scripts/install-mongodb-chart.sh

set -euo pipefail

RELEASE_NAME="production-mongodb"
NAMESPACE="mongodb"
ENVIRONMENT="prod"

echo "🚀 Installing MongoDB Chart..."

# Dry-run pour validation
echo "📋 Validating chart..."
helm install "$RELEASE_NAME" ./mongodb-custom \
  --namespace "$NAMESPACE" \
  --create-namespace \
  --values "mongodb-custom/values-${ENVIRONMENT}.yaml" \
  --dry-run --debug

# Installation réelle
echo "💿 Installing..."
helm install "$RELEASE_NAME" ./mongodb-custom \
  --namespace "$NAMESPACE" \
  --create-namespace \
  --values "mongodb-custom/values-${ENVIRONMENT}.yaml" \
  --wait \
  --timeout 10m

# Afficher le status
helm status "$RELEASE_NAME" -n "$NAMESPACE"

# Lancer les tests
echo "🧪 Running tests..."
helm test "$RELEASE_NAME" -n "$NAMESPACE"

echo "✅ Installation completed successfully!"
```

### Upgrade

```bash
#!/bin/bash
# scripts/upgrade-mongodb-chart.sh

set -euo pipefail

RELEASE_NAME="production-mongodb"
NAMESPACE="mongodb"
ENVIRONMENT="prod"

echo "🔄 Upgrading MongoDB Chart..."

# Vérifier les différences
echo "📊 Checking differences..."
helm diff upgrade "$RELEASE_NAME" ./mongodb-custom \
  --namespace "$NAMESPACE" \
  --values "mongodb-custom/values-${ENVIRONMENT}.yaml" \
  --allow-unreleased

# Confirmer l'upgrade
read -p "Proceed with upgrade? (y/n) " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
  echo "Upgrade cancelled"
  exit 1
fi

# Upgrade avec backup automatique (via pre-upgrade hook)
helm upgrade "$RELEASE_NAME" ./mongodb-custom \
  --namespace "$NAMESPACE" \
  --values "mongodb-custom/values-${ENVIRONMENT}.yaml" \
  --wait \
  --timeout 15m

echo "✅ Upgrade completed successfully!"
```

### Rollback

```bash
#!/bin/bash
# scripts/rollback-mongodb-chart.sh

set -euo pipefail

RELEASE_NAME="production-mongodb"
NAMESPACE="mongodb"

echo "⏮️  Rolling back MongoDB Chart..."

# Afficher l'historique
echo "📜 Release history:"
helm history "$RELEASE_NAME" -n "$NAMESPACE"

# Obtenir la dernière révision réussie
PREVIOUS_REVISION=$(helm history "$RELEASE_NAME" -n "$NAMESPACE" -o json | \
  jq -r '[.[] | select(.status=="deployed")] | sort_by(.revision) | .[-2].revision')

echo "Rolling back to revision: $PREVIOUS_REVISION"

# Rollback
helm rollback "$RELEASE_NAME" "$PREVIOUS_REVISION" \
  --namespace "$NAMESPACE" \
  --wait \
  --timeout 10m

echo "✅ Rollback completed successfully!"
```

---

## Chart Repository

### Packaging et Distribution

```bash
#!/bin/bash
# scripts/package-chart.sh

set -euo pipefail

CHART_DIR="mongodb-custom"
REPO_DIR="chart-repo"

echo "📦 Packaging MongoDB Chart..."

# Linter le chart
echo "🔍 Linting chart..."
helm lint "$CHART_DIR"

# Packager le chart
echo "📦 Creating package..."
helm package "$CHART_DIR" --destination "$REPO_DIR"

# Générer l'index
echo "📚 Generating repository index..."
helm repo index "$REPO_DIR" --url https://charts.company.com

echo "✅ Chart packaged successfully!"
echo "📄 Package: $(ls -t $REPO_DIR/mongodb-custom-*.tgz | head -1)"
```

### Chart Museum

```yaml
# chartmuseum-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chartmuseum
  namespace: chartmuseum
spec:
  replicas: 2
  selector:
    matchLabels:
      app: chartmuseum
  template:
    metadata:
      labels:
        app: chartmuseum
    spec:
      containers:
        - name: chartmuseum
          image: ghcr.io/helm/chartmuseum:v0.16.0

          args:
            - --port=8080
            - --storage=local
            - --storage-local-rootdir=/charts
            - --depth=1

          ports:
            - containerPort: 8080

          volumeMounts:
            - name: charts
              mountPath: /charts

          resources:
            limits:
              cpu: 500m
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 128Mi

      volumes:
        - name: charts
          persistentVolumeClaim:
            claimName: chartmuseum-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: chartmuseum
  namespace: chartmuseum
spec:
  type: ClusterIP
  selector:
    app: chartmuseum
  ports:
    - port: 8080
      targetPort: 8080
```

---

## Best Practices

### Checklist Helm Chart

```yaml
# helm-chart-best-practices.yaml
---
structure:
  - ✅ Chart.yaml complet avec métadonnées
  - ✅ README.md avec documentation
  - ✅ values.yaml avec valeurs par défaut sensées
  - ✅ _helpers.tpl avec fonctions réutilisables
  - ✅ NOTES.txt avec instructions post-install

templating:
  - ✅ Utiliser les helpers pour éviter la duplication
  - ✅ Valider les valeurs avec include "mongodb.validateValues"
  - ✅ Supporter les overrides (nameOverride, fullnameOverride)
  - ✅ Utiliser des conditions (if, with, range)
  - ✅ Indentation correcte avec nindent

configuration:
  - ✅ Values par environnement (dev, staging, prod)
  - ✅ Secrets gérés via existing secrets
  - ✅ Resources configurables
  - ✅ Affinity, tolerations, nodeSelector configurables
  - ✅ Service account et RBAC

lifecycle:
  - ✅ Hooks pre-install, post-install, pre-upgrade
  - ✅ Tests Helm intégrés
  - ✅ Annotations helm.sh/hook appropriées
  - ✅ TTL et delete policies

documentation:
  - ✅ Commentaires dans values.yaml
  - ✅ NOTES.txt détaillé
  - ✅ README avec exemples d'utilisation
  - ✅ CHANGELOG pour suivre les versions

versioning:
  - ✅ Semantic versioning pour Chart.version
  - ✅ AppVersion correspond à la version de MongoDB
  - ✅ CHANGELOG à jour
  - ✅ Git tags pour releases

testing:
  - ✅ helm lint sans erreurs
  - ✅ helm template génère des manifests valides
  - ✅ helm test avec tests de connexion
  - ✅ Chart validé avec chart-testing (ct)

security:
  - ✅ Pas de credentials hardcodés
  - ✅ Utiliser des secrets Kubernetes
  - ✅ SecurityContext défini
  - ✅ Resources limits définis
  - ✅ RBAC minimal nécessaire
```

---

## Conclusion

Les Helm Charts fournissent une solution puissante pour packager et déployer MongoDB sur Kubernetes :

**Avantages clés :**
- **Reproductibilité** : Déploiements cohérents entre environnements
- **Versioning** : Gestion des versions et rollback facilités
- **Templating** : Configuration flexible par environnement
- **Lifecycle** : Hooks pour automatisation (backup, init, etc.)
- **Distribution** : Partage via repositories
- **Testing** : Validation automatisée des déploiements

**Points d'attention :**
- Maintenir la compatibilité entre versions de chart
- Documenter clairement les breaking changes
- Tester les upgrades et rollbacks
- Sécuriser les secrets et credentials
- Valider les values avant déploiement

Un chart bien conçu réduit significativement la complexité opérationnelle et améliore la fiabilité des déploiements MongoDB sur Kubernetes.

---


⏭️ [Ansible pour MongoDB](/18-devops-deploiement/06-ansible-mongodb.md)
