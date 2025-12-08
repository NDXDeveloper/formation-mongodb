🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.2 Création d'un Cluster Atlas

## Introduction

La création d'un cluster Atlas est bien plus qu'un simple clic sur "Deploy". C'est une décision architecturale qui impacte la **performance, la disponibilité, la conformité, et les coûts** pour toute la durée de vie de votre application. Cette section guide les architectes cloud et les équipes DevOps à travers les choix critiques du provisionnement d'un cluster.

### 🎯 Décisions Clés

Lors de la création d'un cluster, vous devez prendre des décisions sur :

1. **Organisation et Projet** : Structure hiérarchique et isolation
2. **Cloud Provider** : AWS, Azure, ou GCP
3. **Région(s)** : Localisation géographique et multi-région
4. **Type de Cluster** : Serverless, Shared, ou Dedicated
5. **Tier/Taille** : Dimensionnement (RAM, vCPU, storage)
6. **Configuration** : Réplication, backup, auto-scaling, sécurité

---

## 📋 Workflow de Provisionnement

### Vue d'Ensemble du Processus

```
┌─────────────────────────────────────────────────────────────────────┐
│              FLUX DE CRÉATION D'UN CLUSTER ATLAS                    │
└─────────────────────────────────────────────────────────────────────┘

1. ORGANISATION                                    [Temps: 2 min]
   ┌──────────────────────────────────────────────────────────┐
   │ • Création ou sélection organisation                     │
   │ • Configuration billing                                  │
   │ • Définition des admins                                  │
   └──────────────────────────────────────────────────────────┘
                            ↓
2. PROJET                                          [Temps: 1 min]
   ┌──────────────────────────────────────────────────────────┐
   │ • Création projet (Dev/Staging/Prod)                     │
   │ • Configuration des membres et rôles                     │
   │ • API Keys pour automation                               │
   └──────────────────────────────────────────────────────────┘
                            ↓
3. CLUSTER CONFIGURATION                           [Temps: 5 min]
   ┌──────────────────────────────────────────────────────────┐
   │ • Choix du provider (AWS/Azure/GCP)                      │
   │ • Sélection région(s)                                    │
   │ • Type de cluster (Serverless/Shared/Dedicated)          │
   │ • Sizing (M0 → M700+)                                    │
   │ • Version MongoDB                                        │
   │ • Options avancées (backup, auto-scaling)                │
   └──────────────────────────────────────────────────────────┘
                            ↓
4. NETWORK & SECURITY                              [Temps: 3 min]
   ┌──────────────────────────────────────────────────────────┐
   │ • IP Access List (whitelisting)                          │
   │ • VPC Peering / Private Endpoint (optionnel)             │
   │ • Database Users (création premier user)                 │
   └──────────────────────────────────────────────────────────┘
                            ↓
5. PROVISIONING                                    [Temps: 3-7 min]
   ┌──────────────────────────────────────────────────────────┐
   │ • Allocation infrastructure cloud                        │
   │ • Déploiement replica set                                │
   │ • Configuration réplication                              │
   │ • Initialisation backups                                 │
   │ • Activation monitoring                                  │
   └──────────────────────────────────────────────────────────┘
                            ↓
6. READY TO CONNECT                                [Total: ~15 min]
   ┌──────────────────────────────────────────────────────────┐
   │ ✅ Cluster opérationnel                                  │
   │ 🔗 Connection string disponible                          │
   │ 📊 Monitoring actif                                      │
   │ 💾 Backups configurés                                    │
   └──────────────────────────────────────────────────────────┘
```

---

## 🏢 Étape 1 : Organisation et Projet

### Hiérarchie Organisationnelle

Atlas utilise une hiérarchie à trois niveaux pour organiser les ressources :

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HIÉRARCHIE ATLAS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ORGANISATION (Niveau Entreprise)                                   │
│  ├─ Billing Account                                                 │
│  ├─ Organization Admins                                             │
│  ├─ API Keys (org-level)                                            │
│  ├─ Support Plan                                                    │
│  │                                                                  │
│  └─► PROJET 1: Production                                           │
│       ├─ Project Settings                                           │
│       ├─ Access Manager (RBAC)                                      │
│       ├─ API Keys (project-level)                                   │
│       ├─ Integrations (monitoring, alerting)                        │
│       │                                                             │
│       ├─► CLUSTER: prod-primary-cluster                             │
│       │    ├─ M60 Dedicated                                         │
│       │    ├─ AWS us-east-1                                         │
│       │    ├─ 3-node replica set                                    │
│       │    └─ Sharded (4 shards)                                    │
│       │                                                             │
│       ├─► CLUSTER: prod-analytics-cluster                           │
│       │    ├─ M40 Dedicated                                         │
│       │    ├─ AWS us-east-1                                         │
│       │    └─ Analytics nodes                                       │
│       │                                                             │
│       └─► DATA LAKE: prod-data-lake                                 │
│            └─ S3 federated queries                                  │
│                                                                     │
│  └─► PROJET 2: Staging                                              │
│       └─► CLUSTER: staging-cluster (M30)                            │
│                                                                     │
│  └─► PROJET 3: Development                                          │
│       ├─► CLUSTER: dev-cluster (M10)                                │
│       └─► CLUSTER: feature-x-cluster (Serverless)                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Stratégies d'Organisation

#### Option A : Une Organisation, Projets par Environnement

```
MyCompany Organization
  ├─ Project: Production
  ├─ Project: Staging
  ├─ Project: Development
  └─ Project: QA/Testing
```

**Avantages** :
- Facturation unifiée
- Gestion centralisée
- Partage simple des ressources

**Inconvénients** :
- Moins de séparation entre environnements
- Risque de "blast radius" en cas d'erreur

#### Option B : Organisations Séparées par Environnement (High-Security)

```
MyCompany-Prod Organization
  └─ Project: Production-Primary

MyCompany-NonProd Organization
  ├─ Project: Staging
  ├─ Project: Development
  └─ Project: Testing
```

**Avantages** :
- Isolation maximale
- Conformité stricte (HIPAA, PCI-DSS)
- Pas de risque d'impact cross-environment

**Inconvénients** :
- Facturation séparée
- Overhead administratif
- Duplication des configurations

#### Option C : Par Business Unit (Entreprise)

```
MyCompany Organization
  ├─ Project: BU-Retail
  ├─ Project: BU-Wholesale
  ├─ Project: BU-International
  └─ Project: Shared-Services
```

### Configuration de l'Organisation

**Éléments à configurer** :

```yaml
Organization Configuration:
  # Identité
  name: "MyCompany Inc."

  # Facturation
  billing:
    method: "credit_card" | "invoice"
    contact: "billing@mycompany.com"
    payment_method: "automatic"

  # Support
  support_plan: "basic" | "developer" | "production" | "enterprise"

  # Admins
  organization_owners:
    - email: "cto@mycompany.com"
    - email: "devops-lead@mycompany.com"

  # API Keys (pour IaC)
  api_keys:
    - name: "terraform-provisioning"
      roles: ["ORG_OWNER"]
    - name: "ci-cd-pipeline"
      roles: ["ORG_MEMBER"]

  # Préférences
  preferences:
    multi_factor_auth_required: true
    require_ip_access_list: true
    restrict_employee_access: false
```

---

## ☁️ Étape 2 : Choix du Cloud Provider

Atlas supporte trois fournisseurs cloud majeurs. Le choix dépend de plusieurs facteurs.

### Comparaison des Providers

```
┌───────────────────────────────────────────────────────────────────────┐
│              AWS vs AZURE vs GCP - ATLAS PERSPECTIVE                  │
├───────────────────────────────────────────────────────────────────────┤
│
│  CRITÈRE               AWS           AZURE         GCP
│  ─────────────────────────────────────────────────────────────────
│  Régions Disponibles   60+           40+           30+
│  Adoption Atlas        ~70%          ~20%          ~10%
│  Maturité Features     ⭐⭐⭐⭐⭐      ⭐⭐⭐⭐☆       ⭐⭐⭐⭐☆
│  Private Link          ✅            ✅             ✅
│  VPC Peering           ✅            ✅             ✅
│  Encryption (BYOK)     ✅ KMS        ✅ Key Vault  ✅ Cloud KMS
│  Instance Types        Nombreux     Nombreux      Limités
│  Network Perf          Excellent    Excellent     Excellent
│  Pricing               $$$          $$$           $$
│  Atlas Features        Complet      Complet       Complet
│
└───────────────────────────────────────────────────────────────────────┘
```

### Critères de Décision

#### 1. Infrastructure Existante

```
┌─────────────────────────────────────────────────────────────┐
│         SI VOTRE INFRASTRUCTURE EST DÉJÀ SUR :              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  AWS      → Choisir AWS Atlas                               │
│             • VPC Peering simplifié                         │
│             • PrivateLink pour zéro exposition Internet     │
│             • IAM intégration                               │
│             • Même région = latence minimale                │
│                                                             │
│  Azure    → Choisir Azure Atlas                             │
│             • VNet Peering                                  │
│             • Private Endpoint                              │
│             • Azure AD intégration                          │
│             • Réseau interne Microsoft                      │
│                                                             │
│  GCP      → Choisir GCP Atlas                               │
│             • VPC Peering                                   │
│             • Private Service Connect                       │
│             • IAM intégration                               │
│             • Réseau Google backbone                        │
│                                                             │
│  Multi    → Stratégie multi-cloud Atlas                     │
│             • Disaster Recovery cross-cloud                 │
│             • Vendor diversification                        │
│             • Complexité réseau accrue                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2. Conformité et Data Residency

```
RÉGION                  PROVIDER RECOMMANDÉ           RATIONALE
─────────────────────────────────────────────────────────────────
US (général)            AWS us-east-1                 Maturité, options
US (gouvernement)       AWS GovCloud*                 FedRAMP
Europe (GDPR)           AWS eu-west-1 (Dublin)        Large disponibilité
                        ou eu-central-1 (Frankfurt)
Asie-Pacifique          AWS ap-southeast-1            Singapore hub
Chine                   Alibaba Cloud*                Conformité locale
Inde                    AWS ap-south-1 (Mumbai)       Data localization
Australie               AWS ap-southeast-2 (Sydney)   APRA compliance
Moyen-Orient            AWS me-south-1 (Bahrain)      Data sovereignty

* Support limité ou en cours
```

#### 3. Latence et Performance

**Règle d'or** : Déployer Atlas dans la **même région** que vos application servers.

```
┌────────────────────────────────────────────────────────────────┐
│                    IMPACT DE LA LATENCE                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Application (us-east-1) → MongoDB (us-east-1)                 │
│     Latency: ~1-2ms                                            │
│     Throughput: Optimal                                        │
│     ✅ Configuration idéale                                    │
│                                                                │
│  Application (us-east-1) → MongoDB (eu-west-1)                 │
│     Latency: ~80-100ms                                         │
│     Throughput: -50%                                           │
│     ⚠️ Performance degradée                                    │
│                                                                │
│  Application (us-east-1) → MongoDB (ap-southeast-1)            │
│     Latency: ~200-250ms                                        │
│     Throughput: -75%                                           │
│     ❌ Inacceptable pour OLTP                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### 4. Coûts

Les coûts varient légèrement entre providers :

```
EXEMPLE: Cluster M30 (8GB RAM, 40GB Storage)

Provider    Region          Monthly Cost    Data Transfer Out
─────────────────────────────────────────────────────────────────
AWS         us-east-1       $285           $0.09/GB
AWS         eu-west-1       $310           $0.09/GB
AWS         ap-south-1      $320           $0.13/GB

Azure       East US         $290           $0.087/GB
Azure       West Europe     $315           $0.087/GB

GCP         us-central1     $275           $0.12/GB
GCP         europe-west1    $300           $0.12/GB

* Prix indicatifs, vérifier pricing actuel
```

### Architecture Multi-Cloud

Pour la haute disponibilité et le disaster recovery :

```
┌──────────────────────────────────────────────────────────────────┐
│              ARCHITECTURE MULTI-CLOUD ATLAS                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CLUSTER: production-multi-cloud                                 │
│                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐    │
│  │     AWS      │      │    AZURE     │      │     GCP      │    │
│  │  us-east-1   │      │   East US    │      │ us-central1  │    │
│  │              │      │              │      │              │    │
│  │  [Primary]   │◄────►│ [Secondary]  │◄────►│ [Secondary]  │    │
│  │   Electable  │      │  Electable   │      │  Electable   │    │
│  │              │      │              │      │              │    │
│  └──────────────┘      └──────────────┘      └──────────────┘    │
│                                                                  │
│  AVANTAGES:                                                      │
│  • Pas de single point of failure (cloud provider)               │
│  • Disaster recovery cross-cloud                                 │
│  • Vendor lock-in minimisé                                       │
│                                                                  │
│  INCONVÉNIENTS:                                                  │
│  • Latence inter-cloud (2-10ms)                                  │
│  • Coûts data transfer                                           │
│  • Complexité réseau (peering multi-cloud)                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🌍 Étape 3 : Sélection de la Région

### Critères de Sélection

#### 1. Proximité Utilisateurs / Applications

```
┌──────────────────────────────────────────────────────────────────┐
│                  ARCHITECTURE GÉOGRAPHIQUE                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cas 1: APPLICATION MONO-RÉGION                                  │
│                                                                  │
│         Users (US) ───► App Servers (us-east-1)                  │
│                              │                                   │
│                              ▼                                   │
│                         MongoDB Atlas                            │
│                         (us-east-1)                              │
│                                                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━     │
│                                                                  │
│  Cas 2: APPLICATION MULTI-RÉGION (GLOBAL)                        │
│                                                                  │
│  Users (US) ───► App Servers (us-east-1) ──┐                     │
│                                            │                     │
│  Users (EU) ───► App Servers (eu-west-1) ──┼──► Load Balancer    │
│                                            │         │           │
│  Users (APAC) ─► App Servers (ap-south-1) ─┘         │           │
│                                                      ▼           │
│                                             ┌──────────────────┐ │
│                                             │ MongoDB Cluster  │ │
│                                             │   Multi-Region   │ │
│                                             │  (3+ régions)    │ │
│                                             └──────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### 2. Data Residency et Conformité

```yaml
# Exemple: Configuration GDPR-compliant
cluster:
  name: "eu-production-cluster"

  # Données STRICTEMENT en Europe
  region_config:
    priority_1:
      provider: "AWS"
      region: "EU_WEST_1"        # Irlande
      electable: true

    priority_2:
      provider: "AWS"
      region: "EU_CENTRAL_1"     # Allemagne
      electable: true

    priority_3:
      provider: "AZURE"
      region: "EUROPE_NORTH"     # Finlande
      electable: true

  # Backup AUSSI en Europe
  backup_region: "EU_WEST_1"

  # Pas de lecture depuis hors UE
  read_preference: "primaryPreferred"

  labels:
    - key: "data-classification"
      value: "personal-data"
    - key: "compliance"
      value: "gdpr"
```

### Configurations Multi-Région

#### Configuration 1 : High Availability (Same Region)

```
┌───────────────────────────────────────────────────────┐
│          CLUSTER: HA MONO-RÉGION (AWS us-east-1)      │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Availability Zone 1a    Availability Zone 1b         │
│  ┌──────────────┐        ┌──────────────┐             │
│  │   Primary    │◄──────►│  Secondary   │             │
│  │  (Electable) │        │ (Electable)  │             │
│  └──────────────┘        └──────────────┘             │
│                                                       │
│                  Availability Zone 1c                 │
│                  ┌──────────────┐                     │
│                  │  Secondary   │                     │
│                  │ (Electable)  │                     │
│                  └──────────────┘                     │
│                                                       │
│  RPO: ~0 (synchronous within region)                  │
│  RTO: ~30 seconds (automatic failover)                │
│  Latence: <2ms entre AZs                              │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**Cas d'usage** : Applications critiques mono-région nécessitant HA.

#### Configuration 2 : Disaster Recovery (Multi-Region)

```
┌───────────────────────────────────────────────────────────────────┐
│              CLUSTER: DISASTER RECOVERY MULTI-RÉGION              │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PRIMARY REGION                SECONDARY REGION                   │
│  us-east-1 (Virginie)          us-west-2 (Oregon)                 │
│                                                                   │
│  ┌────────────────┐            ┌────────────────┐                 │
│  │   Primary      │            │   Secondary    │                 │
│  │   (Priority 7) │◄──────────►│   (Priority 6) │                 │
│  └────────────────┘            └────────────────┘                 │
│          │                              │                         │
│          ▼                              ▼                         │
│  ┌────────────────┐            ┌────────────────┐                 │
│  │   Secondary    │            │   Secondary    │                 │
│  │   (Priority 5) │            │   (Priority 4) │                 │
│  └────────────────┘            └────────────────┘                 │
│                                                                   │
│                      ARBITER (Optional)                           │
│                      eu-west-1 (Irlande)                          │
│                      ┌────────────────┐                           │
│                      │    Arbiter     │                           │
│                      │   (No data)    │                           │
│                      └────────────────┘                           │
│                                                                   │
│  Scénario Normal:    Primary = us-east-1                          │
│  Scénario Disaster:  Primary = us-west-2 (failover auto)          │
│  RPO: ~5-10 secondes (async replication)                          │
│  RTO: ~2-3 minutes (detection + élection)                         │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**Cas d'usage** : Continuité d'activité en cas de défaillance régionale complète.

#### Configuration 3 : Global Distribution

```
┌───────────────────────────────────────────────────────────────────┐
│                 CLUSTER: DISTRIBUTION GLOBALE                     │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  AMERICAS             EUROPE               ASIA-PACIFIC           │
│  us-east-1            eu-west-1            ap-southeast-1         │
│                                                                   │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐           │
│  │ Primary  │◄───────►│Secondary │◄───────►│Secondary │           │
│  │ Priority │         │ Priority │         │ Priority │           │
│  │    7     │         │    6     │         │    5     │           │
│  └──────────┘         └──────────┘         └──────────┘           │
│       │                    │                     │                │
│       ▼                    ▼                     ▼                │
│  [Analytics]          [Analytics]          [Analytics]            │
│   Nodes                Nodes                Nodes                 │
│                                                                   │
│  READ PREFERENCE PAR RÉGION:                                      │
│  • US users      → nearest (us-east-1)                            │
│  • EU users      → nearest (eu-west-1)                            │
│  • APAC users    → nearest (ap-southeast-1)                       │
│                                                                   │
│  WRITES: Toujours vers Primary (us-east-1)                        │
│  Latence Write: Variable selon localisation user                  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**Cas d'usage** : Applications globales avec utilisateurs distribués géographiquement.

---

## 🎛️ Étape 4 : Type de Cluster

Atlas propose trois types de clusters selon vos besoins.

### Comparaison des Types

```
┌────────────────────────────────────────────────────────────────────────┐
│              SERVERLESS vs SHARED vs DEDICATED                         │
├────────────────────────────────────────────────────────────────────────┤
│
│  CRITÈRE           SERVERLESS    SHARED (M0-M5)   DEDICATED (M10+)
│  ────────────────────────────────────────────────────────────────────
│  Pricing Model     Pay-per-op    Monthly fixed   Monthly fixed
│  Min Cost          $0            $0 (M0)          $57/mo (M10)
│  Auto-Scaling      ♾️  Infini     ❌ Non           ✅ Oui
│  Max Size          1TB           5GB              100TB+
│  Shared Infra      ❌ Non        ✅ Oui (M0-M5)   ❌ Non
│  Performance       Variable      Limité           Garanti
│  Backup            ✅ Auto       ⚠️ Basique (M2+) ✅ PITR
│  VPC Peering       ❌ Non        ❌ Non            ✅ Oui
│  Private Endpoint  ❌ Non        ❌ Non            ✅ Oui
│  Atlas Search      ✅ Oui        ❌ Non            ✅ Oui
│  Vector Search     ✅ Oui        ❌ Non            ✅ Oui
│  Multi-Region      ❌ Non        ❌ Non            ✅ Oui
│  Analytics Nodes   ❌ Non        ❌ Non            ✅ Oui
│  Sharding          ❌ Non        ❌ Non            ✅ Oui (M30+)
│  Monitoring        Basic         Basic            Advanced
│  Support SLA       Basic         Basic            Production/Ent.
│
│  USE CASE          Dev/Test      Learning/MVP     Production
│                    Spiky loads   Small apps       Enterprise
│
└────────────────────────────────────────────────────────────────────────┘
```

### Type 1 : Serverless Instances

```
┌───────────────────────────────────────────────────────────┐
│                  SERVERLESS ARCHITECTURE                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  CARACTÉRISTIQUES:                                        │
│  • Scaling automatique 0 → ∞                              │
│  • Pas de provisioning                                    │
│  • Pay-per-operation                                      │
│  • Auto-pause si idle (0 coût)                            │
│                                                           │
│  PRICING MODEL:                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Reads:   $0.10 per million                          │  │
│  │ Writes:  $1.00 per million                          │  │
│  │ Storage: $0.25/GB-month                             │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  EXEMPLE DE COÛT:                                         │
│  • 10M reads/month   = $1.00                              │
│  • 1M writes/month   = $1.00                              │
│  • 10GB storage      = $2.50                              │
│  • Total:              $4.50/month                        │
│                                                           │
│  IDEAL POUR:                                              │
│  ✅ Prototypes et MVPs                                    │
│  ✅ Workloads intermittents                               │
│  ✅ Applications avec trafic variable                     │
│  ✅ Dev/Test environments                                 │
│                                                           │
│  PAS RECOMMANDÉ POUR:                                     │
│  ❌ Workloads prévisibles et constants                    │
│  ❌ Multi-région                                          │
│  ❌ VPC Peering requis                                    │
│  ❌ Très haute performance (latency SLA)                  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Scaling Behavior** :

```
Traffic Pattern              Serverless Response
─────────────────────────────────────────────────────────────

00:00-08:00  [▁▁▁▁▁▁▁▁]     Auto-scales DOWN → minimal cost
08:00-10:00  [▃▃▅▅▇▇▇▇]     Scales UP gradually
10:00-18:00  [███████████]   Peak capacity
18:00-24:00  [▇▇▅▅▃▃▁▁]     Scales DOWN
Idle periods [         ]     Auto-PAUSE → $0 compute cost
```

### Type 2 : Shared Clusters (M0, M2, M5)

```
┌────────────────────────────────────────────────────────────┐
│                   SHARED CLUSTERS TIERS                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  M0 (FREE TIER)                                            │
│  ├─ RAM:        512 MB                                     │
│  ├─ Storage:    512 MB                                     │
│  ├─ vCPU:       Shared                                     │
│  ├─ Transfer:   Bandwidth limits                           │
│  ├─ Backups:    ❌ Non                                     │
│  ├─ Cost:       $0 forever                                 │
│  └─ Use:        Learning, prototypes, demos                │
│                                                            │
│  M2 (SHARED - PAID)                                        │
│  ├─ RAM:        2 GB                                       │
│  ├─ Storage:    2 GB                                       │
│  ├─ vCPU:       Shared                                     │
│  ├─ Transfer:   Higher limits                              │
│  ├─ Backups:    ✅ Basic                                   │
│  ├─ Cost:       ~$9/month                                  │
│  └─ Use:        Small hobby projects                       │
│                                                            │
│  M5 (SHARED - PAID)                                        │
│  ├─ RAM:        5 GB                                       │
│  ├─ Storage:    5 GB                                       │
│  ├─ vCPU:       Shared                                     │
│  ├─ Transfer:   Higher limits                              │
│  ├─ Backups:    ✅ Basic                                   │
│  ├─ Cost:       ~$25/month                                 │
│  └─ Use:        Personal projects, small apps              │
│                                                            │
│  ⚠️ LIMITATIONS SHARED:                                    │
│  • Infrastructure partagée (noisy neighbors)               │
│  • Pas de VPC peering                                      │
│  • Pas d'analytics nodes                                   │
│  • Pas de sharding                                         │
│  • Support communautaire uniquement                        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Type 3 : Dedicated Clusters (M10+)

```
┌───────────────────────────────────────────────────────────────────┐
│                   DEDICATED CLUSTERS SIZING                       │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  TIER    RAM     vCPU   STORAGE   IOPS      MONTHLY    USE CASE   │
│  ──────────────────────────────────────────────────────────────   │
│  M10     2GB     2      10-128GB  3,000     $57        Dev/Test   │
│  M20     4GB     2      10-256GB  3,000     $140       Small Prod │
│  M30     8GB     2      10-512GB  3,000     $285       Prod       │
│  M40     16GB    4      10-1TB    6,000     $630       Prod+      │
│  M50     32GB    8      10-4TB    16,000    $1,525     Enterprise │
│  M60     64GB    16     10-4TB    16,000    $3,050     Enterprise │
│  M80     128GB   32     10-4TB    16,000    $6,480     Large      │
│  M140    192GB   48     10-4TB    25,000    $10,800    Very Large │
│  M200    256GB   64     10-4TB    50,000    $14,400    Massive    │
│  M300    384GB   96     10-4TB    50,000    $23,040    Massive    │
│  M400+   Custom  Custom Custom    Custom    Custom     Ultra      │
│                                                                   │
│  * Prix AWS us-east-1, peut varier selon région/provider          │
│  ** Storage et IOPS peuvent être augmentés indépendamment         │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**Architecture d'un Cluster Dedicated** :

```
┌───────────────────────────────────────────────────────────┐
│           CLUSTER DEDICATED M40 (3-node replica set)      │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  NODE 1 (Primary)         NODE 2 (Secondary)              │
│  ┌──────────────────┐     ┌──────────────────┐            │
│  │ Instance: M40    │     │ Instance: M40    │            │
│  │ RAM: 16GB        │◄───►│ RAM: 16GB        │            │
│  │ vCPU: 4          │     │ vCPU: 4          │            │
│  │ Storage: 250GB   │     │ Storage: 250GB   │            │
│  │ IOPS: 6,000      │     │ IOPS: 6,000      │            │
│  └──────────────────┘     └──────────────────┘            │
│          ▲                                                │
│          │                                                │
│          ▼                                                │
│  NODE 3 (Secondary)                                       │
│  ┌──────────────────┐                                     │
│  │ Instance: M40    │                                     │
│  │ RAM: 16GB        │                                     │
│  │ vCPU: 4          │                                     │
│  │ Storage: 250GB   │                                     │
│  │ IOPS: 6,000      │                                     │
│  └──────────────────┘                                     │
│                                                           │
│  FEATURES INCLUDED:                                       │
│  ✅ VPC Peering                                           │
│  ✅ Private Endpoints                                     │
│  ✅ Continuous Backup (PITR)                              │
│  ✅ Atlas Search                                          │
│  ✅ Performance Advisor                                   │
│  ✅ Custom Roles                                          │
│  ✅ 24/7 Production Support (payant)                      │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## ⚙️ Étape 5 : Configuration Avancée

### Configuration de Base

```yaml
cluster_configuration:
  name: "production-cluster"

  # Provider et Région
  provider:
    name: "AWS"
    instance_size: "M40"
    region: "US_EAST_1"

  # Version MongoDB
  mongo_db_version: "7.0"

  # Replica Set
  replication:
    num_shards: 1                    # 1 = replica set, 2+ = sharded
    replication_factor: 3            # 3 nodes
    electable_nodes: 3               # Tous peuvent être Primary
    analytics_nodes: 0               # Nœuds analytics optionnels
    read_only_nodes: 0               # Read-only optionnels

  # Auto-Scaling
  auto_scaling:
    disk:
      enabled: true
    compute:
      enabled: true
      min_instance_size: "M40"
      max_instance_size: "M60"
```

### Auto-Scaling Configuration

```
┌────────────────────────────────────────────────────────────────────┐
│                    AUTO-SCALING POLICIES                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DISK AUTO-SCALING                                                 │
│  ─────────────────                                                 │
│  Trigger: Disk usage > 90%                                         │
│  Action:  Add 10-20% more storage                                  │
│  Max:     4TB per node (tier dependent)                            │
│  Cooldown: 6 hours between scale-ups                               │
│                                                                    │
│  Example:                                                          │
│  250GB → 275GB → 300GB → 350GB → 400GB                             │
│                                                                    │
│  ───────────────────────────────────────────────────────────────── │
│                                                                    │
│  COMPUTE AUTO-SCALING (Vertical)                                   │
│  ────────────────────────────────                                  │
│  Trigger: CPU/RAM sustained > 75% for 1 hour                       │
│  Action:  Scale up to next tier                                    │
│  Range:   M40 ↔ M50 ↔ M60                                          │
│  Cooldown: Configurable (default: 1 hour)                          │
│  Downscale: Only if < 50% for 24+ hours                            │
│                                                                    │
│  Example Scenario:                                                 │
│  M40 (16GB) → Traffic spike → M50 (32GB)                           │
│            → Traffic normal → M40 (after 24h low usage)            │
│                                                                    │
│  ⚠️ Note: Brief downtime (~30-60s) during compute scaling          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Backup Configuration

```yaml
backup_configuration:
  # Cloud Backup (Continuous)
  cloud_backup:
    enabled: true

    # Snapshot Schedule
    snapshot_schedule:
      - frequency: "HOURLY"
        retention: 48              # Keep 48 hourly snapshots

      - frequency: "DAILY"
        retention: 7               # Keep 7 daily snapshots

      - frequency: "WEEKLY"
        retention: 4               # Keep 4 weekly snapshots

      - frequency: "MONTHLY"
        retention: 12              # Keep 12 monthly snapshots

    # Point-in-Time Recovery
    pit_enabled: true
    pit_window_hours: 72           # PITR for last 72 hours

    # Cross-Region Backup
    copy_to_regions:
      - "US_WEST_2"                # DR copy in different region

  # Legacy Backup (if needed)
  legacy_backup:
    enabled: false
```

### Monitoring and Alerting

```yaml
monitoring:
  # Profiling
  profiler:
    enabled: true
    slow_query_threshold_ms: 100

  # Metrics Granularity
  metrics_granularity: "1_MINUTE"  # 1_MINUTE or 1_HOUR

alerts:
  # CPU Alert
  - event_type: "HOST_CPU_USAGE"
    enabled: true
    threshold:
      operator: "GREATER_THAN"
      value: 80
      units: "RAW"
    notifications:
      - type: "EMAIL"
        email_address: "ops-team@mycompany.com"
      - type: "PAGERDUTY"
        service_key: "xxxxx"

  # Disk Space Alert
  - event_type: "HOST_DISK_USAGE"
    enabled: true
    threshold:
      operator: "GREATER_THAN"
      value: 85
      units: "RAW"
    notifications:
      - type: "SLACK"
        channel_name: "#alerts-production"
```

---

## 🎯 Guide de Décision : Quel Tier Choisir ?

### Matrice de Décision

```
┌──────────────────────────────────────────────────────────────────────┐
│                     TIER SELECTION MATRIX                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  YOUR PROFILE                    RECOMMENDED TIER                    │
│  ────────────────────────────────────────────────────────────────    │
│  Learning MongoDB                M0 (Free)                           │
│  Personal project                M2/M5 (Shared) or Serverless        │
│  Startup MVP                     Serverless or M10                   │
│  Dev/Test environment            M10-M20                             │
│  Small production                M20-M30                             │
│  Production (< 100GB)            M30-M40                             │
│  Production (100GB-500GB)        M40-M60                             │
│  Production (500GB-2TB)          M60-M80                             │
│  Production (2TB-10TB)           M80-M200 + Sharding                 │
│  Production (10TB+)              M200+ + Multi-Shard                 │
│  Enterprise (very large)         M300+ + Custom                      │
│  High IOPS requirements          M140+ (25K-50K IOPS)                │
│  Analytics workload              M40+ with Analytics Nodes           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Calcul de Sizing

**Méthodologie** :

```
1. ESTIMER LA TAILLE DES DONNÉES
   ───────────────────────────────
   Working Set Size = Données actives + Indexes

   Formule simplifiée:
   RAM nécessaire ≈ Working Set Size × 1.5

   Exemple:
   - Documents: 50GB
   - Indexes: 10GB
   - Working Set: 60GB
   - RAM requise: 90GB
   → Choisir M80 (128GB RAM)

2. ESTIMER LE THROUGHPUT
   ──────────────────────
   Operations/seconde:
   - Reads: X req/s
   - Writes: Y req/s

   Règle générale M40:
   - ~10,000 reads/sec
   - ~5,000 writes/sec

   Si besoin > capacité → Tier supérieur ou Sharding

3. ESTIMER LE STOCKAGE
   ────────────────────
   Croissance = Current Size × Growth Rate × Time

   Exemple:
   - Taille actuelle: 200GB
   - Croissance: 20GB/mois
   - Horizon: 12 mois
   - Total: 200 + (20 × 12) = 440GB
   → Choisir tier avec 500GB+ (M40-M60)
   → Activer auto-scaling disk
```

---

## 🚀 Provisionnement : Méthodes

### Méthode 1 : Atlas UI (Console Web)

**Avantages** : Rapide, visuel, découverte des options
**Inconvénients** : Pas reproductible, pas versionné

```
Steps:
1. Login → Atlas Console
2. Organizations → Select Organization
3. Projects → Create/Select Project
4. Clusters → Build a Cluster
5. Choose:
   - Provider (AWS/Azure/GCP)
   - Region
   - Cluster Type (Serverless/Shared/Dedicated)
   - Tier Size
6. Configure:
   - Additional Settings (backup, auto-scaling)
   - Network Access (IP whitelist)
   - Database Users
7. Create Cluster (3-7 minutes)
8. Get Connection String
```

### Méthode 2 : Terraform (Infrastructure as Code)

**Avantages** : Reproductible, versionné, CI/CD
**Inconvénients** : Courbe d'apprentissage

```hcl
# Provider configuration
terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.14"
    }
  }
}

provider "mongodbatlas" {
  public_key  = var.atlas_public_key
  private_key = var.atlas_private_key
}

# Create Project
resource "mongodbatlas_project" "production" {
  name   = "production"
  org_id = var.atlas_org_id
}

# Create Cluster
resource "mongodbatlas_cluster" "production_cluster" {
  project_id = mongodbatlas_project.production.id
  name       = "production-cluster"

  # Provider Settings
  provider_name               = "AWS"
  provider_region_name        = "US_EAST_1"
  provider_instance_size_name = "M40"

  # MongoDB Version
  mongo_db_major_version = "7.0"

  # Auto-Scaling
  auto_scaling_disk_gb_enabled = true
  auto_scaling_compute_enabled = true
  auto_scaling_compute_scale_down_enabled = true

  # Backup
  cloud_backup                    = true
  pit_enabled                     = true

  # Replica Set Configuration
  replication_specs {
    num_shards = 1

    regions_config {
      region_name     = "US_EAST_1"
      electable_nodes = 3
      priority        = 7
      read_only_nodes = 0
    }
  }

  # Advanced Configuration
  advanced_configuration {
    javascript_enabled           = true
    minimum_enabled_tls_protocol = "TLS1_2"
  }

  # Tags
  labels {
    key   = "environment"
    value = "production"
  }

  labels {
    key   = "team"
    value = "backend"
  }
}

# Output connection string
output "connection_string" {
  value     = mongodbatlas_cluster.production_cluster.connection_strings[0].standard_srv
  sensitive = true
}
```

### Méthode 3 : Atlas CLI

```bash
# Login
atlas auth login

# Create project
atlas projects create "production" \
  --orgId "5f2e3c4d5e6f7a8b9c0d1e2f"

# Create cluster
atlas clusters create "production-cluster" \
  --projectId "6g3f4d5e6f7a8b9c0d1e2f3g" \
  --provider AWS \
  --region US_EAST_1 \
  --tier M40 \
  --mdbVersion 7.0 \
  --diskSizeGB 250 \
  --backup \
  --tag environment=production \
  --tag team=backend

# Get cluster details
atlas clusters describe "production-cluster" \
  --projectId "6g3f4d5e6f7a8b9c0d1e2f3g"

# Get connection string
atlas clusters connectionStrings describe "production-cluster" \
  --projectId "6g3f4d5e6f7a8b9c0d1e2f3g"
```

---

## ✅ Checklist Post-Création

Après la création du cluster, vérifier :

```
┌────────────────────────────────────────────────────────────┐
│           POST-DEPLOYMENT CHECKLIST                        │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ☐ Cluster provisionné et opérationnel                     │
│  ☐ Replica Set healthy (3/3 nodes UP)                      │
│  ☐ Backups activés et premier snapshot créé                │
│  ☐ Database users créés avec principes du moindre privilège│
│  ☐ IP Access List configurée (pas de 0.0.0.0/0 en prod)    │
│  ☐ Connection string testé depuis application              │
│  ☐ Monitoring dashboards vérifiés                          │
│  ☐ Alertes configurées (CPU, RAM, Disk, Connections)       │
│  ☐ VPC Peering / Private Endpoint configuré (si applicable)│
│  ☐ Encryption at rest validé                               │
│  ☐ TLS/SSL activé et forcé                                 │
│  ☐ Indexes créés sur les collections                       │
│  ☐ Performance Advisor activé                              │
│  ☐ Labels/Tags appliqués (environment, team, cost-center)  │
│  ☐ Budget alerts configurées                               │
│  ☐ Documentation mise à jour                               │
│  ☐ Runbook disaster recovery créé                          │
│  ☐ Accès équipe configuré (RBAC project)                   │
│  ☐ Conformité validée (data residency, encryption)         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé

La création d'un cluster Atlas implique des **décisions architecturales critiques** :

### Points Clés à Retenir

1. **Hiérarchie** : Organisation → Projet → Cluster
   - Utiliser les projets pour isoler les environnements

2. **Provider** : AWS (70%), Azure (20%), GCP (10%)
   - Choisir selon infrastructure existante et conformité

3. **Région** : Proximité > Tout
   - Déployer dans la région de vos app servers
   - Multi-région pour DR et HA

4. **Type de Cluster** :
   - **Serverless** : Dev/test, workloads variables
   - **Shared** : Learning, prototypes, petits projets
   - **Dedicated** : Production, tout sérieux

5. **Sizing** : Commencer petit, scaler ensuite
   - Working Set Size guide le choix de RAM
   - Auto-scaling pour l'élasticité
   - Monitoring continu pour ajuster

6. **Configuration** :
   - Activer backups (PITR en production)
   - Auto-scaling disk et compute
   - Alerting proactif

7. **Méthode** :
   - UI pour exploration
   - **Terraform pour production** (IaC, reproductible)
   - CLI pour automation

### Temps de Provisionnement

- **Serverless** : ~2 minutes
- **Shared** : ~3 minutes
- **Dedicated** : ~5-7 minutes
- **Multi-région** : ~10-15 minutes

### Coûts Typiques (Production)

- **Small** (M20-M30) : $140-285/mois
- **Medium** (M40-M60) : $630-3,050/mois
- **Large** (M80-M200) : $6,480-14,400/mois
- **+ Data transfer** : $0.09-0.13/GB
- **+ Backups** : inclus

---


⏭️ [Tiers gratuit (M0) et options payantes](/14-mongodb-atlas/03-tiers-gratuit-options-payantes.md)
