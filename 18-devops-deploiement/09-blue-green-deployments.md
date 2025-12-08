🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.9 Déploiements Blue/Green et Canary

## Introduction

Les déploiements Blue/Green et Canary sont des stratégies essentielles pour atteindre des déploiements sans interruption (zero-downtime) tout en minimisant les risques. Ces approches permettent de valider progressivement les nouvelles versions avant de les exposer à l'ensemble du trafic de production.

```
┌─────────────────────────────────────────────────────────────────────┐
│               Deployment Strategies Comparison                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Rolling Update:                                                    │
│  ┌────┬────┬────┬────┐     ┌────┬────┬────┬────┐                    │
│  │ v1 │ v1 │ v1 │ v1 │ --> │ v1 │ v1 │ v1 │ v2 │ --> ...            │
│  └────┴────┴────┴────┘     └────┴────┴────┴────┘                    │
│  • Gradual replacement                                              │
│  • Minimal resource usage                                           │
│  • Risk: two versions coexist                                       │
│                                                                     │
│  Blue/Green:                                                        │
│  ┌────────────────┐         ┌────────────────┐                      │
│  │  BLUE (v1)     │ ◄────   │  GREEN (v2)    │                      │
│  │  ┌──┬──┬──┬──┐ │  100%   │  ┌──┬──┬──┬──┐ │                      │
│  │  │v1│v1│v1│v1│ │         │  │v2│v2│v2│v2│ │                      │
│  │  └──┴──┴──┴──┘ │         │  └──┴──┴──┴──┘ │                      │
│  └────────────────┘         └────────────────┘                      │
│         │                            │                              │
│         └────── Switch traffic ──────┘                              │
│  • Full environment duplication                                     │
│  • Instant rollback                                                 │
│  • Double resources needed                                          │
│                                                                     │
│  Canary:                                                            │
│  ┌────────────────────────────────────────────────┐                 │
│  │  Production (v1)                               │                 │
│  │  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┐                  │                 │
│  │  │v1│v1│v1│v1│v1│v1│v1│v1│v1│ 90%              │                 │
│  │  └──┴──┴──┴──┴──┴──┴──┴──┴──┘                  │                 │
│  │                                                │                 │
│  │  Canary (v2)                                   │                 │
│  │  ┌──┐                                          │                 │
│  │  │v2│ 10%                                      │                 │
│  │  └──┘                                          │                 │
│  └────────────────────────────────────────────────┘                 │
│  • Progressive rollout (10% → 50% → 100%)                           │
│  • Real user traffic validation                                     │
│  • Minimal risk exposure                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Avantages et Inconvénients

```yaml
# deployment-strategies-comparison.yaml
---
blue_green:
  advantages:
    - "Zero-downtime deployment"
    - "Instant rollback capability"
    - "Full testing before traffic switch"
    - "Single version active at a time"
    - "Clear separation of environments"

  disadvantages:
    - "Double infrastructure cost during deployment"
    - "Database migrations complexity"
    - "Large initial resource requirement"
    - "All-or-nothing approach"

  best_for:
    - "Critical applications"
    - "When instant rollback is crucial"
    - "Predictable traffic patterns"
    - "Applications with strict version control"

canary:
  advantages:
    - "Progressive risk mitigation"
    - "Real user feedback early"
    - "Minimal resource overhead"
    - "Gradual validation"
    - "Easy abort on issues"

  disadvantages:
    - "Complex traffic routing"
    - "Longer deployment time"
    - "Need robust monitoring"
    - "Two versions coexist longer"

  best_for:
    - "High-risk changes"
    - "User-facing applications"
    - "When metrics are reliable"
    - "Microservices architectures"

rolling:
  advantages:
    - "Minimal resource usage"
    - "Gradual deployment"
    - "Built-in most orchestrators"

  disadvantages:
    - "Slower rollback"
    - "Two versions coexist"
    - "Harder to test fully"

  best_for:
    - "Stateless applications"
    - "Internal services"
    - "Limited resources"
```

---

## Architecture Blue/Green

### Principe de Fonctionnement

```
┌────────────────────────────────────────────────────────────────────┐
│            Blue/Green Deployment with MongoDB                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Load Balancer                             │  │
│  │              (NGINX / HAProxy / Ingress)                     │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                        │
│                  ┌────────┴────────┐                               │
│                  │                 │                               │
│                  ▼                 ▼                               │
│  ┌───────────────────────┐  ┌───────────────────────┐              │
│  │   BLUE Environment    │  │  GREEN Environment    │              │
│  │      (Active)         │  │     (Standby)         │              │
│  │                       │  │                       │              │
│  │  ┌─────────────────┐  │  │  ┌─────────────────┐  │              │
│  │  │  App v1.0       │  │  │  │  App v2.0       │  │              │
│  │  │  ┌───┬───┬───┐  │  │  │  │  ┌───┬───┬───┐  │  │              │
│  │  │  │Pod│Pod│Pod│  │  │  │  │  │Pod│Pod│Pod│  │  │              │
│  │  │  └───┴───┴───┘  │  │  │  │  └───┴───┴───┘  │  │              │
│  │  └─────────────────┘  │  │  └─────────────────┘  │              │
│  │          │            │  │          │            │              │
│  │          ▼            │  │          ▼            │              │
│  │  ┌─────────────────┐  │  │  ┌─────────────────┐  │              │
│  │  │  Service Blue   │  │  │  │  Service Green  │  │              │
│  │  └─────────────────┘  │  │  └─────────────────┘  │              │
│  └───────────────────────┘  └───────────────────────┘              │
│                  │                      │                          │
│                  └──────────┬───────────┘                          │
│                             │                                      │
│                             ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Shared MongoDB Cluster                          │  │
│  │                                                              │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │  │
│  │  │ Primary  │  │Secondary │  │Secondary │                    │  │
│  │  │  Node    │  │  Node    │  │  Node    │                    │  │
│  │  └──────────┘  └──────────┘  └──────────┘                    │  │
│  │                                                              │  │
│  │  • Both environments connect to same cluster                 │  │
│  │  • Migrations run before traffic switch                      │  │
│  │  • Backward compatible changes essential                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Implémentation Kubernetes

### Namespace et Services

```yaml
# blue-green-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
---
# Blue Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-blue
  namespace: production
  labels:
    app: myapp
    version: blue
spec:
  type: ClusterIP
  selector:
    app: myapp
    version: blue
  ports:
    - name: http
      port: 80
      targetPort: 3000
---
# Green Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-green
  namespace: production
  labels:
    app: myapp
    version: green
spec:
  type: ClusterIP
  selector:
    app: myapp
    version: green
  ports:
    - name: http
      port: 80
      targetPort: 3000
---
# Production Service (points to active version)
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
  annotations:
    active-version: "blue"  # Tracks which version is active
spec:
  type: ClusterIP
  selector:
    app: myapp
    version: blue  # Initially points to blue
  ports:
    - name: http
      port: 80
      targetPort: 3000
  sessionAffinity: ClientIP  # Optional: maintain session stickiness
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

### Deployments Blue et Green

```yaml
# blue-green-deployments.yaml
---
# Blue Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: myapp
      version: blue

  template:
    metadata:
      labels:
        app: myapp
        version: blue
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"

    spec:
      serviceAccountName: myapp

      # Anti-affinity to spread pods
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - myapp
                    - key: version
                      operator: In
                      values:
                        - blue
                topologyKey: kubernetes.io/hostname

      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0
          imagePullPolicy: Always

          ports:
            - name: http
              containerPort: 3000
            - name: metrics
              containerPort: 9090

          env:
            - name: VERSION
              value: "blue"

            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-uri
                  key: connection-string

            - name: NODE_ENV
              value: "production"

            - name: LOG_LEVEL
              value: "info"

          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 30
---
# Green Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
  labels:
    app: myapp
    version: green
spec:
  replicas: 0  # Initially scaled to 0
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: myapp
      version: green

  template:
    metadata:
      labels:
        app: myapp
        version: green
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"

    spec:
      serviceAccountName: myapp

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - myapp
                    - key: version
                      operator: In
                      values:
                        - green
                topologyKey: kubernetes.io/hostname

      containers:
        - name: myapp
          image: myregistry/myapp:v2.0.0  # New version
          imagePullPolicy: Always

          ports:
            - name: http
              containerPort: 3000
            - name: metrics
              containerPort: 9090

          env:
            - name: VERSION
              value: "green"

            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-uri
                  key: connection-string

            - name: NODE_ENV
              value: "production"

            - name: LOG_LEVEL
              value: "info"

            # Feature flags for gradual rollout
            - name: ENABLE_NEW_FEATURE
              value: "true"

          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 30
```

### Ingress avec Traffic Splitting

```yaml
# blue-green-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # NGINX specific annotations for blue/green
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"

    # Custom headers to identify version
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Backend-Version $service_version always;
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: myapp-tls

  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp  # Points to active service (blue or green)
                port:
                  number: 80
---
# Alternative: Split traffic using weights (NGINX Plus or Istio)
# For Istio VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-traffic-split
  namespace: production
spec:
  hosts:
    - app.example.com

  gateways:
    - myapp-gateway

  http:
    - match:
        - uri:
            prefix: /

      route:
        # Blue environment (current production)
        - destination:
            host: myapp-blue
            port:
              number: 80
          weight: 100  # 100% traffic to blue initially

        # Green environment (new version)
        - destination:
            host: myapp-green
            port:
              number: 80
          weight: 0    # 0% traffic to green initially

      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure,refused-stream

      timeout: 30s
```

---

## Script de Déploiement Blue/Green

### Script Bash Complet

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh
# Blue/Green deployment script for Kubernetes

set -euo pipefail

# Configuration
NAMESPACE="${NAMESPACE:-production}"
APP_NAME="${APP_NAME:-myapp}"
NEW_VERSION="${NEW_VERSION:-}"
KUBECTL="${KUBECTL:-kubectl}"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Functions
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Get current active version
get_active_version() {
    local selector=$($KUBECTL get service ${APP_NAME} -n ${NAMESPACE} -o jsonpath='{.spec.selector.version}' 2>/dev/null || echo "blue")
    echo "$selector"
}

# Get inactive version
get_inactive_version() {
    local active=$(get_active_version)
    if [ "$active" = "blue" ]; then
        echo "green"
    else
        echo "blue"
    fi
}

# Check if deployment exists
deployment_exists() {
    local version=$1
    $KUBECTL get deployment ${APP_NAME}-${version} -n ${NAMESPACE} &>/dev/null
}

# Scale deployment
scale_deployment() {
    local version=$1
    local replicas=$2

    log_info "Scaling ${APP_NAME}-${version} to ${replicas} replicas..."
    $KUBECTL scale deployment ${APP_NAME}-${version} -n ${NAMESPACE} --replicas=${replicas}
}

# Wait for deployment to be ready
wait_for_deployment() {
    local version=$1
    local timeout=${2:-300}

    log_info "Waiting for ${APP_NAME}-${version} to be ready (timeout: ${timeout}s)..."

    if ! $KUBECTL wait --for=condition=available \
        --timeout=${timeout}s \
        deployment/${APP_NAME}-${version} \
        -n ${NAMESPACE}; then
        log_error "Deployment ${APP_NAME}-${version} failed to become ready"
        return 1
    fi

    log_info "Deployment ${APP_NAME}-${version} is ready"
    return 0
}

# Run health checks
health_check() {
    local version=$1
    local service_name="${APP_NAME}-${version}"

    log_info "Running health checks on ${service_name}..."

    # Get one pod from the deployment
    local pod=$($KUBECTL get pods -n ${NAMESPACE} \
        -l app=${APP_NAME},version=${version} \
        -o jsonpath='{.items[0].metadata.name}')

    if [ -z "$pod" ]; then
        log_error "No pods found for ${service_name}"
        return 1
    fi

    # Execute health check
    local health_status=$($KUBECTL exec -n ${NAMESPACE} ${pod} -- \
        curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)

    if [ "$health_status" != "200" ]; then
        log_error "Health check failed with status: ${health_status}"
        return 1
    fi

    log_info "Health check passed"
    return 0
}

# Run smoke tests
smoke_tests() {
    local version=$1
    local service_url="http://${APP_NAME}-${version}.${NAMESPACE}.svc.cluster.local"

    log_info "Running smoke tests on ${service_url}..."

    # Create a test pod
    local test_pod="smoke-test-$$"

    $KUBECTL run ${test_pod} -n ${NAMESPACE} \
        --image=curlimages/curl:latest \
        --restart=Never \
        --rm -i --quiet \
        -- sh -c "
            # Test health endpoint
            echo 'Testing /health endpoint...'
            curl -f ${service_url}/health || exit 1

            # Test API endpoint
            echo 'Testing /api/version endpoint...'
            curl -f ${service_url}/api/version || exit 1

            # Test database connection
            echo 'Testing /api/db/ping endpoint...'
            curl -f ${service_url}/api/db/ping || exit 1

            echo 'All smoke tests passed'
        " 2>&1

    local exit_code=$?

    if [ $exit_code -eq 0 ]; then
        log_info "Smoke tests passed"
        return 0
    else
        log_error "Smoke tests failed"
        return 1
    fi
}

# Switch traffic to new version
switch_traffic() {
    local target_version=$1

    log_info "Switching traffic to ${target_version}..."

    # Update service selector
    $KUBECTL patch service ${APP_NAME} -n ${NAMESPACE} \
        --type='json' \
        -p="[{\"op\": \"replace\", \"path\": \"/spec/selector/version\", \"value\": \"${target_version}\"}]"

    # Update annotation to track active version
    $KUBECTL annotate service ${APP_NAME} -n ${NAMESPACE} \
        active-version=${target_version} --overwrite

    log_info "Traffic switched to ${target_version}"
}

# Monitor deployment after switch
monitor_deployment() {
    local version=$1
    local duration=${2:-300}  # 5 minutes

    log_info "Monitoring ${version} for ${duration} seconds..."

    local start_time=$(date +%s)
    local end_time=$((start_time + duration))

    while [ $(date +%s) -lt $end_time ]; do
        # Check pod status
        local ready_pods=$($KUBECTL get pods -n ${NAMESPACE} \
            -l app=${APP_NAME},version=${version} \
            -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -o True | wc -l)

        local total_pods=$($KUBECTL get pods -n ${NAMESPACE} \
            -l app=${APP_NAME},version=${version} \
            --no-headers | wc -l)

        if [ $ready_pods -ne $total_pods ]; then
            log_error "Some pods are not ready: ${ready_pods}/${total_pods}"
            return 1
        fi

        # Check error rate (would integrate with monitoring system)
        # For now, just verify health endpoint
        if ! health_check ${version}; then
            log_error "Health check failed during monitoring"
            return 1
        fi

        log_info "Monitoring: ${ready_pods}/${total_pods} pods ready"
        sleep 30
    done

    log_info "Monitoring completed successfully"
    return 0
}

# Rollback to previous version
rollback() {
    local target_version=$1

    log_warn "Rolling back to ${target_version}..."

    switch_traffic ${target_version}

    log_info "Rollback completed"
}

# Cleanup old version
cleanup_old_version() {
    local version=$1

    log_info "Cleaning up ${APP_NAME}-${version}..."

    scale_deployment ${version} 0

    log_info "Cleanup completed"
}

# Main deployment function
deploy() {
    if [ -z "$NEW_VERSION" ]; then
        log_error "NEW_VERSION environment variable is required"
        exit 1
    fi

    log_info "Starting Blue/Green deployment..."
    log_info "Namespace: ${NAMESPACE}"
    log_info "Application: ${APP_NAME}"
    log_info "New Version: ${NEW_VERSION}"

    # Get current state
    local active_version=$(get_active_version)
    local inactive_version=$(get_inactive_version)

    log_info "Active version: ${active_version}"
    log_info "Target version: ${inactive_version}"

    # Step 1: Update inactive deployment with new version
    log_info "Step 1: Updating ${inactive_version} deployment..."

    if ! deployment_exists ${inactive_version}; then
        log_error "Deployment ${APP_NAME}-${inactive_version} does not exist"
        exit 1
    fi

    # Update image
    $KUBECTL set image deployment/${APP_NAME}-${inactive_version} \
        ${APP_NAME}=myregistry/${APP_NAME}:${NEW_VERSION} \
        -n ${NAMESPACE}

    # Step 2: Scale up inactive deployment
    log_info "Step 2: Scaling up ${inactive_version} deployment..."
    scale_deployment ${inactive_version} 3

    # Step 3: Wait for deployment to be ready
    log_info "Step 3: Waiting for ${inactive_version} to be ready..."
    if ! wait_for_deployment ${inactive_version} 600; then
        log_error "Deployment failed"
        cleanup_old_version ${inactive_version}
        exit 1
    fi

    # Step 4: Health checks
    log_info "Step 4: Running health checks..."
    if ! health_check ${inactive_version}; then
        log_error "Health checks failed"
        cleanup_old_version ${inactive_version}
        exit 1
    fi

    # Step 5: Smoke tests
    log_info "Step 5: Running smoke tests..."
    if ! smoke_tests ${inactive_version}; then
        log_error "Smoke tests failed"
        cleanup_old_version ${inactive_version}
        exit 1
    fi

    # Step 6: Switch traffic
    log_info "Step 6: Switching traffic to ${inactive_version}..."
    switch_traffic ${inactive_version}

    # Step 7: Monitor new version
    log_info "Step 7: Monitoring ${inactive_version}..."
    if ! monitor_deployment ${inactive_version} 300; then
        log_error "Monitoring detected issues, rolling back..."
        rollback ${active_version}
        exit 1
    fi

    # Step 8: Cleanup old version
    log_info "Step 8: Cleaning up ${active_version}..."
    cleanup_old_version ${active_version}

    log_info "Blue/Green deployment completed successfully!"
    log_info "Active version is now: ${inactive_version}"
}

# Parse command line arguments
case "${1:-deploy}" in
    deploy)
        deploy
        ;;

    rollback)
        ROLLBACK_VERSION="${2:-}"
        if [ -z "$ROLLBACK_VERSION" ]; then
            ROLLBACK_VERSION=$(get_inactive_version)
        fi
        rollback "$ROLLBACK_VERSION"
        ;;

    status)
        ACTIVE=$(get_active_version)
        echo "Active version: $ACTIVE"
        $KUBECTL get deployments -n ${NAMESPACE} -l app=${APP_NAME}
        ;;

    *)
        echo "Usage: $0 {deploy|rollback|status}"
        exit 1
        ;;
esac
```

---

## Canary Deployment

### Architecture Canary

```yaml
# canary-deployment.yaml
---
# Stable Deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
  namespace: production
spec:
  replicas: 9  # 90% of capacity
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
        version: v1.0.0
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0
          ports:
            - containerPort: 3000
---
# Canary Deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
  namespace: production
spec:
  replicas: 1  # 10% of capacity
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v2.0.0  # New version
          ports:
            - containerPort: 3000
          env:
            - name: TRACK
              value: "canary"
---
# Service (load balances across both)
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp  # Selects both stable and canary
  ports:
    - port: 80
      targetPort: 3000
```

### Istio VirtualService pour Canary

```yaml
# istio-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-canary
  namespace: production
spec:
  hosts:
    - myapp.example.com

  gateways:
    - myapp-gateway

  http:
    # Canary routing based on headers (for testing)
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: myapp
            subset: canary
          weight: 100

    # Production traffic split
    - route:
        # Stable version
        - destination:
            host: myapp
            subset: stable
          weight: 90

        # Canary version
        - destination:
            host: myapp
            subset: canary
          weight: 10

      # Retry configuration
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure

      # Timeout
      timeout: 30s

      # Circuit breaker
      connectionPool:
        tcp:
          maxConnections: 100
        http:
          http1MaxPendingRequests: 50
          http2MaxRequests: 100
          maxRequestsPerConnection: 2
---
# DestinationRule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
  namespace: production
spec:
  host: myapp

  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST

    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2

    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50

  subsets:
    - name: stable
      labels:
        track: stable

    - name: canary
      labels:
        track: canary
```

### Script Canary Progressif

```bash
#!/bin/bash
# scripts/canary-deploy.sh
# Progressive canary deployment

set -euo pipefail

NAMESPACE="${NAMESPACE:-production}"
APP_NAME="${APP_NAME:-myapp}"
NEW_VERSION="${NEW_VERSION:-}"

# Canary progression stages
CANARY_STAGES=(10 25 50 75 100)
MONITOR_DURATION=300  # 5 minutes per stage

log_info() {
    echo "[INFO] $1"
}

log_error() {
    echo "[ERROR] $1"
}

# Get metrics from Prometheus
get_error_rate() {
    local track=$1

    # Query Prometheus for error rate
    local query="rate(http_requests_total{app=\"${APP_NAME}\",track=\"${track}\",status=~\"5..\"}[5m]) / rate(http_requests_total{app=\"${APP_NAME}\",track=\"${track}\"}[5m]) * 100"

    local error_rate=$(curl -s "http://prometheus:9090/api/v1/query?query=${query}" | \
        jq -r '.data.result[0].value[1] // "0"')

    echo "$error_rate"
}

get_latency_p99() {
    local track=$1

    local query="histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{app=\"${APP_NAME}\",track=\"${track}\"}[5m]))"

    local latency=$(curl -s "http://prometheus:9090/api/v1/query?query=${query}" | \
        jq -r '.data.result[0].value[1] // "0"')

    echo "$latency"
}

# Update traffic split
update_traffic_split() {
    local canary_weight=$1
    local stable_weight=$((100 - canary_weight))

    log_info "Updating traffic split: stable=${stable_weight}%, canary=${canary_weight}%"

    kubectl patch virtualservice ${APP_NAME}-canary -n ${NAMESPACE} --type='json' -p="[
        {\"op\": \"replace\", \"path\": \"/spec/http/1/route/0/weight\", \"value\": ${stable_weight}},
        {\"op\": \"replace\", \"path\": \"/spec/http/1/route/1/weight\", \"value\": ${canary_weight}}
    ]"
}

# Scale canary deployment
scale_canary() {
    local percentage=$1
    local total_replicas=10
    local canary_replicas=$((total_replicas * percentage / 100))
    local stable_replicas=$((total_replicas - canary_replicas))

    log_info "Scaling: stable=${stable_replicas}, canary=${canary_replicas}"

    kubectl scale deployment ${APP_NAME}-stable -n ${NAMESPACE} --replicas=${stable_replicas}
    kubectl scale deployment ${APP_NAME}-canary -n ${NAMESPACE} --replicas=${canary_replicas}
}

# Monitor canary health
monitor_canary() {
    local duration=$1

    log_info "Monitoring canary for ${duration} seconds..."

    local start_time=$(date +%s)
    local end_time=$((start_time + duration))

    while [ $(date +%s) -lt $end_time ]; do
        # Get metrics
        local canary_error_rate=$(get_error_rate "canary")
        local stable_error_rate=$(get_error_rate "stable")
        local canary_latency=$(get_latency_p99 "canary")
        local stable_latency=$(get_latency_p99 "stable")

        log_info "Metrics - Canary: ${canary_error_rate}% errors, ${canary_latency}s p99 | Stable: ${stable_error_rate}% errors, ${stable_latency}s p99"

        # Check if canary is performing worse
        local error_threshold=5
        local latency_threshold_multiplier=1.5

        if (( $(echo "$canary_error_rate > $error_threshold" | bc -l) )); then
            log_error "Canary error rate too high: ${canary_error_rate}%"
            return 1
        fi

        if (( $(echo "$canary_latency > $stable_latency * $latency_threshold_multiplier" | bc -l) )); then
            log_error "Canary latency too high: ${canary_latency}s vs ${stable_latency}s"
            return 1
        fi

        sleep 30
    done

    log_info "Monitoring passed"
    return 0
}

# Deploy canary
deploy_canary() {
    log_info "Starting canary deployment..."

    # Deploy canary with new version
    kubectl set image deployment/${APP_NAME}-canary \
        ${APP_NAME}=myregistry/${APP_NAME}:${NEW_VERSION} \
        -n ${NAMESPACE}

    kubectl wait --for=condition=available \
        deployment/${APP_NAME}-canary \
        -n ${NAMESPACE} \
        --timeout=300s

    # Progressive rollout
    for stage in "${CANARY_STAGES[@]}"; do
        log_info "Stage: ${stage}% canary traffic"

        # Update traffic split
        update_traffic_split ${stage}

        # Scale deployments
        scale_canary ${stage}

        # Wait for scaling
        sleep 30

        # Monitor
        if ! monitor_canary ${MONITOR_DURATION}; then
            log_error "Canary deployment failed at ${stage}% stage"
            rollback_canary
            exit 1
        fi

        log_info "Stage ${stage}% completed successfully"
    done

    # Finalize: make canary the new stable
    log_info "Promoting canary to stable..."

    kubectl set image deployment/${APP_NAME}-stable \
        ${APP_NAME}=myregistry/${APP_NAME}:${NEW_VERSION} \
        -n ${NAMESPACE}

    # Reset to 100% stable
    update_traffic_split 0
    scale_canary 0

    log_info "Canary deployment completed successfully!"
}

# Rollback canary
rollback_canary() {
    log_info "Rolling back canary deployment..."

    # Set traffic to 100% stable
    update_traffic_split 0

    # Scale canary to 0
    kubectl scale deployment ${APP_NAME}-canary -n ${NAMESPACE} --replicas=0

    # Scale stable back to full
    kubectl scale deployment ${APP_NAME}-stable -n ${NAMESPACE} --replicas=10

    log_info "Rollback completed"
}

# Main
case "${1:-deploy}" in
    deploy)
        deploy_canary
        ;;

    rollback)
        rollback_canary
        ;;

    *)
        echo "Usage: $0 {deploy|rollback}"
        exit 1
        ;;
esac
```

---

## Gestion de la Base de Données

### Stratégies pour MongoDB

```javascript
// database-migration-strategies.js
/**
 * Stratégies de migration de base de données pour Blue/Green
 */

// 1. Backward Compatible Migrations
// Toutes les migrations doivent être compatibles avec l'ancienne version

const backwardCompatibleMigration = async (db) => {
  // Phase 1: Ajouter nouveaux champs (sans supprimer les anciens)
  await db.collection('users').updateMany(
    { newField: { $exists: false } },
    { $set: { newField: null } }
  );

  // Phase 2: Les deux versions utilisent les données
  // - Blue (v1) ignore newField
  // - Green (v2) utilise newField

  // Phase 3: Après que Green soit stable, supprimer oldField
  // (dans une migration ultérieure)
};

// 2. Feature Flags pour changements de schéma
const featureFlaggedSchema = {
  // Document flexible qui supporte les deux versions
  user: {
    _id: ObjectId(),
    email: String,

    // Old format (always present for backward compatibility)
    firstName: String,
    lastName: String,

    // New format (optional, used by v2)
    fullName: String,  // firstName + lastName

    // Feature flag
    schemaVersion: 2
  }
};

// Application code avec feature flags
class UserService {
  async getUser(userId) {
    const user = await db.collection('users').findOne({ _id: userId });

    // Adapter selon la version du schéma
    if (process.env.USE_NEW_SCHEMA === 'true') {
      // Version v2 (Green)
      if (!user.fullName && user.firstName && user.lastName) {
        // Migration lazy
        user.fullName = `${user.firstName} ${user.lastName}`;
        await db.collection('users').updateOne(
          { _id: userId },
          { $set: { fullName: user.fullName, schemaVersion: 2 } }
        );
      }
      return {
        id: user._id,
        email: user.email,
        fullName: user.fullName
      };
    } else {
      // Version v1 (Blue)
      return {
        id: user._id,
        email: user.email,
        firstName: user.firstName,
        lastName: user.lastName
      };
    }
  }
}

// 3. Read-Write Split
// Séparation lecture/écriture pour validation
const readWriteSplit = {
  async write(data) {
    // Écrire dans les deux formats
    await db.collection('users').updateOne(
      { _id: data.id },
      {
        $set: {
          // Old format (pour Blue)
          firstName: data.firstName,
          lastName: data.lastName,

          // New format (pour Green)
          fullName: data.fullName,

          schemaVersion: 2
        }
      }
    );
  },

  async read(id, useNewFormat = false) {
    const user = await db.collection('users').findOne({ _id: id });

    if (useNewFormat) {
      return {
        id: user._id,
        fullName: user.fullName || `${user.firstName} ${user.lastName}`
      };
    } else {
      return {
        id: user._id,
        firstName: user.firstName,
        lastName: user.lastName
      };
    }
  }
};

// 4. Shadow Traffic
// Envoyer les requêtes aux deux versions pour comparaison
const shadowTraffic = async (request) => {
  // Requête primaire (Blue)
  const primaryResponse = await fetch('http://myapp-blue/api/users', {
    method: 'POST',
    body: JSON.stringify(request)
  });

  // Requête shadow (Green) - en parallèle, sans affecter le résultat
  fetch('http://myapp-green/api/users', {
    method: 'POST',
    body: JSON.stringify(request),
    headers: { 'X-Shadow-Traffic': 'true' }
  }).catch(err => {
    // Logger les erreurs mais ne pas affecter la requête primaire
    console.error('Shadow traffic error:', err);
  });

  // Retourner uniquement la réponse primaire
  return primaryResponse;
};

module.exports = {
  backwardCompatibleMigration,
  featureFlaggedSchema,
  UserService,
  readWriteSplit,
  shadowTraffic
};
```

### Pre-Deployment Migration Script

```javascript
// scripts/pre-deployment-migration.js
const { MongoClient } = require('mongodb');

async function preDeploymentMigration() {
  const uri = process.env.MONGODB_URI;
  const client = await MongoClient.connect(uri);
  const db = client.db();

  console.log('Starting pre-deployment migration...');

  try {
    // 1. Vérifier la compatibilité
    console.log('Checking schema compatibility...');
    const incompatibleDocs = await db.collection('users')
      .countDocuments({
        $or: [
          { email: { $exists: false } },
          { email: { $type: 'string', $eq: '' } }
        ]
      });

    if (incompatibleDocs > 0) {
      throw new Error(`Found ${incompatibleDocs} documents with invalid email`);
    }

    // 2. Créer les nouveaux champs (backward compatible)
    console.log('Adding new fields...');
    const result = await db.collection('users').updateMany(
      { fullName: { $exists: false } },
      [{
        $set: {
          fullName: {
            $concat: ['$firstName', ' ', '$lastName']
          },
          migrationVersion: 2,
          migratedAt: new Date()
        }
      }]
    );

    console.log(`Updated ${result.modifiedCount} documents`);

    // 3. Créer les index nécessaires
    console.log('Creating indexes...');
    await db.collection('users').createIndex(
      { fullName: 'text' },
      { name: 'idx_fullname_text' }
    );

    // 4. Valider les modifications
    console.log('Validating migration...');
    const validation = await db.collection('users').aggregate([
      {
        $match: {
          migrationVersion: 2
        }
      },
      {
        $project: {
          isValid: {
            $eq: [
              '$fullName',
              { $concat: ['$firstName', ' ', '$lastName'] }
            ]
          }
        }
      },
      {
        $match: {
          isValid: false
        }
      },
      {
        $count: 'invalidCount'
      }
    ]).toArray();

    if (validation.length > 0 && validation[0].invalidCount > 0) {
      throw new Error(`Found ${validation[0].invalidCount} documents with invalid data`);
    }

    console.log('Migration completed successfully');

  } catch (error) {
    console.error('Migration failed:', error);
    throw error;
  } finally {
    await client.close();
  }
}

// Run migration
if (require.main === module) {
  preDeploymentMigration()
    .then(() => process.exit(0))
    .catch(error => {
      console.error(error);
      process.exit(1);
    });
}

module.exports = preDeploymentMigration;
```

---

## Monitoring et Observabilité

### Métriques Clés

```yaml
# prometheus-rules-blue-green.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: blue-green-deployment-rules
  namespace: production
spec:
  groups:
    - name: blue-green-deployment
      interval: 30s
      rules:
        # Error rate par version
        - record: deployment:error_rate:rate5m
          expr: |
            rate(http_requests_total{status=~"5.."}[5m])
            /
            rate(http_requests_total[5m])
            * 100

        # Latency p99 par version
        - record: deployment:latency:p99
          expr: |
            histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket[5m])
            )

        # Alertes: Error rate trop élevé
        - alert: HighErrorRate
          expr: deployment:error_rate:rate5m > 5
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on {{ $labels.version }}"
            description: "Error rate is {{ $value }}% on {{ $labels.version }}"

        # Alertes: Latency trop élevée
        - alert: HighLatency
          expr: deployment:latency:p99 > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High latency on {{ $labels.version }}"
            description: "P99 latency is {{ $value }}s on {{ $labels.version }}"

        # Alertes: Écart de performance entre versions
        - alert: PerformanceDivergence
          expr: |
            abs(
              deployment:error_rate:rate5m{version="green"}
              -
              deployment:error_rate:rate5m{version="blue"}
            ) > 3
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Performance divergence between blue and green"
            description: "Error rate difference is {{ $value }}%"
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Blue/Green Deployment",
    "panels": [
      {
        "title": "Traffic Distribution",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (version) (rate(http_requests_total[5m]))"
          }
        ]
      },
      {
        "title": "Error Rate by Version",
        "type": "graph",
        "targets": [
          {
            "expr": "deployment:error_rate:rate5m",
            "legendFormat": "{{ version }}"
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
        "title": "Latency (p99) by Version",
        "type": "graph",
        "targets": [
          {
            "expr": "deployment:latency:p99",
            "legendFormat": "{{ version }}"
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
        "title": "Active Pods",
        "type": "stat",
        "targets": [
          {
            "expr": "count by (version) (kube_pod_status_ready{condition=\"true\",namespace=\"production\"})"
          }
        ]
      }
    ]
  }
}
```

---

## Best Practices

### Checklist Blue/Green

```yaml
# blue-green-best-practices.yaml
---
preparation:
  - ✅ Infrastructure capacity doubled (blue + green)
  - ✅ Database migrations backward compatible
  - ✅ Feature flags implemented
  - ✅ Monitoring dashboards configured
  - ✅ Rollback procedure documented
  - ✅ Team trained on deployment process

deployment:
  - ✅ Pre-deployment backup created
  - ✅ Migrations run and validated
  - ✅ Green environment fully tested
  - ✅ Health checks passing
  - ✅ Smoke tests passing
  - ✅ Performance metrics baseline established
  - ✅ Traffic switch performed during low traffic

monitoring:
  - ✅ Error rates monitored in real-time
  - ✅ Latency metrics tracked
  - ✅ Database performance monitored
  - ✅ Resource utilization checked
  - ✅ Alerts configured
  - ✅ On-call team notified

validation:
  - ✅ User acceptance testing in production
  - ✅ Business metrics validated
  - ✅ No regression in functionality
  - ✅ Performance meets SLAs
  - ✅ Security scan passed

cleanup:
  - ✅ Old environment kept for 24-48h
  - ✅ Logs archived
  - ✅ Post-mortem if issues occurred
  - ✅ Documentation updated
  - ✅ Lessons learned documented

database_considerations:
  - ✅ Migrations are reversible
  - ✅ Both versions can coexist
  - ✅ No data loss on rollback
  - ✅ Connection pooling configured
  - ✅ Read/write split if needed
  - ✅ Backup before migration
  - ✅ Point-in-time recovery available
```

---

## Conclusion

Les déploiements Blue/Green et Canary sont essentiels pour des déploiements production fiables avec MongoDB :

**Blue/Green :**
- **Avantages** : Zero-downtime, rollback instantané, testing complet
- **Challenges** : Coût double, migrations complexes
- **Idéal pour** : Applications critiques, déploiements majeurs

**Canary :**
- **Avantages** : Risque minimisé, validation progressive, retour utilisateur
- **Challenges** : Monitoring complexe, plus long
- **Idéal pour** : Changements risqués, nouvelles features

**Points clés :**
- Migrations backward compatible essentielles
- Feature flags pour coexistence
- Monitoring robuste obligatoire
- Procédure de rollback testée
- Documentation complète

**Avec MongoDB :**
- Base partagée entre environnements
- Migrations non-destructives
- Schema versioning
- Lazy migration strategies
- Dual-write period pour transitions

Ces stratégies, combinées avec une automatisation solide et un monitoring proactif, permettent des déploiements fiables avec un risque minimal pour les applications MongoDB en production.

---


⏭️ [Gestion des configurations](/18-devops-deploiement/10-gestion-configurations.md)
