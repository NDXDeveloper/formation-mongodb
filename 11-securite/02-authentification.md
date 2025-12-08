🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2 Authentification MongoDB

## Introduction

L'authentification est le processus par lequel MongoDB vérifie l'identité d'un client avant de lui accorder l'accès aux ressources. C'est la première ligne de défense contre les accès non autorisés et un prérequis obligatoire pour tout déploiement en production. MongoDB supporte plusieurs mécanismes d'authentification pour s'adapter à différents environnements et exigences de sécurité.

## Vue d'Ensemble des Mécanismes d'Authentification

MongoDB offre 5 mécanismes principaux d'authentification :

### 1. SCRAM (Salted Challenge Response Authentication Mechanism)

**Type** : Authentification par mot de passe
**Versions** : SCRAM-SHA-1 (déprécié), SCRAM-SHA-256 (recommandé)
**Disponibilité** : MongoDB Community + Enterprise
**Cas d'usage** : Mécanisme par défaut, applications standards

```
Client                                Server
  |                                     |
  |-------- 1. Demande connexion ------>|
  |                                     |
  |<------- 2. Challenge (salt) --------|
  |                                     |
  |-------- 3. Proof (hash) ----------->|
  |                                     |
  |<------- 4. Vérification ------------|
  |                                     |
  |========= Authentifié ===============|
```

### 2. x.509 Certificate Authentication

**Type** : Authentification par certificat
**Disponibilité** : MongoDB Community + Enterprise
**Cas d'usage** : Sécurité renforcée, pas de gestion de mots de passe, authentification mutuelle

```
Client (avec certificat)              Server (avec CA)
  |                                     |
  |------ 1. Présentation cert -------->|
  |                                     |
  |                     Vérification cert avec CA
  |                     Extraction CN/Subject
  |                                     |
  |<------ 2. Acceptation/Rejet --------|
  |                                     |
  |========= Authentifié ===============|
```

### 3. LDAP (Lightweight Directory Access Protocol)

**Type** : Authentification externe
**Disponibilité** : MongoDB Enterprise uniquement
**Cas d'usage** : Intégration avec Active Directory, annuaire d'entreprise centralisé

```
Client                    MongoDB              LDAP/AD
  |                          |                    |
  |--- Credentials --------->|                    |
  |                          |--- LDAP Bind ----->|
  |                          |                    |
  |                          |<-- Validation -----|
  |                          |                    |
  |<-- Access granted -------|                    |
  |                          |                    |
```

### 4. Kerberos

**Type** : Authentification externe
**Disponibilité** : MongoDB Enterprise uniquement
**Cas d'usage** : Environnements hautement sécurisés, SSO enterprise

```
Client          MongoDB         KDC (Key Distribution Center)
  |               |                        |
  |------ 1. TGT Request ----------------->|
  |<----- 2. TGT (Ticket Granting) --------|
  |                                        |
  |------ 3. Service Ticket Request ------>|
  |<----- 4. Service Ticket ---------------|
  |                                        |
  |--- 5. Service Ticket ------->|         |
  |                              |         |
  |<---- 6. Access granted ------|         |
  |                              |         |
```

### 5. OIDC (OpenID Connect)

**Type** : Authentification externe moderne
**Disponibilité** : MongoDB Atlas, Enterprise 7.0+
**Cas d'usage** : SSO moderne, intégration avec fournisseurs d'identité cloud (Okta, Auth0, Azure AD)

```
Client          MongoDB         Identity Provider (IdP)
  |               |                        |
  |--- 1. Demande auth ---------------->   |
  |<-- 2. Redirect vers IdP -----------    |
  |                                        |
  |--- 3. Authentification --------------->|
  |<-- 4. Token ID/Access -----------------|
  |                                        |
  |--- 5. Token MongoDB ------->|          |
  |                             |          |
  |                    Validation token avec IdP
  |                             |          |
  |<--- 6. Access granted ------|          |
```

## Comparaison des Mécanismes d'Authentification

| Critère | SCRAM-SHA-256 | x.509 | LDAP | Kerberos | OIDC |
|---------|---------------|-------|------|----------|------|
| **Édition** | Community | Community | Enterprise | Enterprise | Enterprise/Atlas |
| **Complexité** | ⭐ Faible | ⭐⭐⭐ Élevée | ⭐⭐ Moyenne | ⭐⭐⭐⭐ Très élevée | ⭐⭐ Moyenne |
| **Sécurité** | ⭐⭐⭐⭐ Bonne | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐ Bonne | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐⭐ Bonne |
| **Gestion mots de passe** | Locale MongoDB | N/A | Centralisée | N/A | Centralisée |
| **Infrastructure requise** | Aucune | PKI/CA | Serveur LDAP/AD | KDC | IdP (Okta, etc.) |
| **SSO** | ❌ Non | ❌ Non | ✅ Oui | ✅ Oui | ✅ Oui |
| **Rotation credentials** | Manuel | Renouvellement cert | Géré par LDAP | Géré par Kerberos | Géré par IdP |
| **Audit centralisé** | MongoDB only | MongoDB only | LDAP + MongoDB | Kerberos + MongoDB | IdP + MongoDB |
| **Performance** | Excellent | Excellent | Bon | Bon | Bon |
| **Cas d'usage typique** | Applications standard | Haute sécurité, M2M | Entreprise existante | Environnements legacy | Cloud-native, SaaS |

## Architecture de l'Authentification MongoDB

### Composants d'Authentification

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENT                              │
│  • Application                                              │
│  • Driver MongoDB                                           │
│  • Credentials (user/password ou certificat)                │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Connection String
                            │ mongodb://user:pass@host/db?authMechanism=...
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    MONGODB SERVER                           │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. Réception de la demande de connexion              │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  2. Identification du mécanisme d'authentification    │  │
│  │     (basé sur authMechanism ou négociation)           │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│             ┌──────────────┴───────────────┐                │
│             ▼                              ▼                │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │  SCRAM / x.509   │          │  LDAP/Kerberos   │         │
│  │  (Local)         │          │  (External)      │         │
│  └──────────────────┘          └──────────────────┘         │
│             │                              │                │
│             ▼                              ▼                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  3. Vérification des credentials                      │  │
│  │     • SCRAM: Hash password avec salt                  │  │
│  │     • x.509: Validation certificat avec CA            │  │
│  │     • LDAP: Bind LDAP                                 │  │
│  │     • Kerberos: Validation ticket                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  4. Recherche utilisateur dans admin.system.users     │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  5. Chargement des rôles et privilèges                │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  6. Création de la session authentifiée               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    ✅ Accès autorisé
```

### Base de Données admin.system.users

Tous les utilisateurs MongoDB sont stockés dans la collection `admin.system.users`, quel que soit le mécanisme d'authentification :

```javascript
// Structure d'un document utilisateur
{
  _id: "admin.myuser",
  userId: UUID("..."),
  user: "myuser",
  db: "admin",

  // Credentials (pour SCRAM uniquement)
  credentials: {
    "SCRAM-SHA-256": {
      iterationCount: 15000,
      salt: "...",
      storedKey: "...",
      serverKey: "..."
    }
  },

  // Rôles
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "dbAdmin", db: "myapp" }
  ],

  // Mécanismes autorisés
  mechanisms: ["SCRAM-SHA-256"],

  // Restrictions d'authentification (optionnel)
  authenticationRestrictions: [
    {
      clientSource: ["10.0.0.0/8"],
      serverAddress: ["10.0.1.100"]
    }
  ]
}
```

## Configuration de Base de l'Authentification

### Activation de l'Authentification

L'authentification est **désactivée par défaut** dans MongoDB. Pour l'activer :

```yaml
# mongod.conf
security:
  authorization: enabled
```

Pour un replica set, l'authentification interne doit également être configurée :

```yaml
# mongod.conf
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile  # Authentification inter-membres
```

### Première Configuration : Bootstrap

**Problème** : Comment créer le premier utilisateur si l'authentification est activée ?

**Solution** : Exception localhost - MongoDB permet une connexion non authentifiée depuis localhost UNIQUEMENT pour créer le premier utilisateur.

**Procédure de bootstrap** :

```bash
# 1. Démarrer MongoDB sans authentification
mongod --config /etc/mongod.conf

# 2. Se connecter via localhost
mongosh

# 3. Créer le premier utilisateur admin
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),  // Demande interactive du password
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" }
  ]
})

# 4. Arrêter MongoDB
db.adminCommand({ shutdown: 1 })

# 5. Activer l'authentification
# Modifier mongod.conf: security.authorization: enabled

# 6. Redémarrer MongoDB
mongod --config /etc/mongod.conf

# 7. Se connecter avec le nouvel utilisateur
mongosh -u admin -p --authenticationDatabase admin
```

**⚠️ IMPORTANT** : L'exception localhost ne fonctionne que :
- Depuis l'interface localhost (127.0.0.1)
- Quand aucun utilisateur n'existe encore dans la base
- Pour créer des utilisateurs uniquement (pas d'autres opérations)

## Concepts Clés de l'Authentification

### 1. authSource

Le paramètre `authSource` spécifie la base de données où l'utilisateur est défini.

```javascript
// Utilisateur défini dans la base 'admin'
mongosh "mongodb://myuser:password@localhost:27017/myapp?authSource=admin"

// Utilisateur défini dans la base 'myapp'
mongosh "mongodb://myuser:password@localhost:27017/myapp?authSource=myapp"

// Pour l'authentification externe (LDAP, Kerberos)
mongosh "mongodb://user@localhost:27017/myapp?authSource=$external"
```

**Règles** :
- Par défaut, `authSource` = base de données de connexion
- Pour les mécanismes externes (LDAP, Kerberos, x.509), `authSource` doit être `$external`
- Recommandation : toujours définir les utilisateurs dans `admin` pour centraliser

### 2. authMechanism

Spécifie explicitement le mécanisme d'authentification à utiliser.

```javascript
// SCRAM-SHA-256 (explicite)
mongodb://user:pass@host/db?authMechanism=SCRAM-SHA-256

// x.509
mongodb://CN=client,OU=IT,O=Company@host/db?authMechanism=MONGODB-X509&authSource=$external

// LDAP
mongodb://user:pass@host/db?authMechanism=PLAIN&authSource=$external

// Kerberos
mongodb://user%40REALM@host/db?authMechanism=GSSAPI&authSource=$external

// OIDC
mongodb://host/db?authMechanism=MONGODB-OIDC&authSource=$external
```

**Négociation automatique** : Si `authMechanism` n'est pas spécifié, MongoDB et le driver négocient automatiquement le mécanisme le plus fort disponible.

### 3. mechanisms (Liste des Mécanismes Autorisés)

Lors de la création d'un utilisateur, vous pouvez restreindre les mécanismes autorisés :

```javascript
db.createUser({
  user: "secureuser",
  pwd: "password",
  roles: ["readWrite"],
  mechanisms: ["SCRAM-SHA-256"]  // N'autorise QUE SCRAM-SHA-256
})

// Par défaut si non spécifié: ["SCRAM-SHA-1", "SCRAM-SHA-256"]
```

**Cas d'usage** :
- Forcer l'utilisation de SCRAM-SHA-256 uniquement
- Interdire les mécanismes legacy (SCRAM-SHA-1)
- Conformité de sécurité

## String de Connexion et Authentification

### Formats de Connection String

#### Format Standard

```
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/[defaultauthdb][?options]]
```

#### Format DNS Seedlist (SRV)

```
mongodb+srv://[username:password@]host[/[defaultauthdb][?options]]
```

### Exemples de Connection Strings

#### SCRAM avec Replica Set

```javascript
// Production - Full URI
const uri = "mongodb://appuser:SecurePass123!@mongo1.internal:27017,mongo2.internal:27017,mongo3.internal:27017/myapp?replicaSet=rs0&authSource=admin&tls=true&tlsCAFile=/etc/ssl/ca.pem";

// Avec options séparées
const uri = "mongodb://appuser:SecurePass123!@mongo1.internal:27017,mongo2.internal:27017,mongo3.internal:27017/myapp";
const options = {
  replicaSet: "rs0",
  authSource: "admin",
  tls: true,
  tlsCAFile: "/etc/ssl/ca.pem",
  maxPoolSize: 50,
  retryWrites: true
};
```

#### x.509 Certificate

```javascript
const uri = "mongodb://CN=app01,OU=IT,O=Company,L=Paris,C=FR@mongo1.internal:27017/myapp?authMechanism=MONGODB-X509&authSource=$external&tls=true";

const options = {
  tlsCertificateKeyFile: "/etc/ssl/client.pem",
  tlsCAFile: "/etc/ssl/ca.pem"
};
```

#### LDAP (Enterprise)

```javascript
const uri = "mongodb://ldapuser:ldappass@mongo.internal:27017/myapp?authMechanism=PLAIN&authSource=$external";
```

### Encodage des Caractères Spéciaux

Les caractères spéciaux dans les passwords doivent être URL-encodés :

```javascript
// Password contient des caractères spéciaux: P@ssw0rd!#$
// Encodé:
const encodedPass = encodeURIComponent("P@ssw0rd!#$");
// Résultat: P%40ssw0rd!%23%24

const uri = `mongodb://user:${encodedPass}@host/db`;
```

**Caractères nécessitant un encodage** :
```
: / ? # [ ] @ ! $ & ' ( ) * + , ; =
```

## Authentification dans les Architectures Distribuées

### Replica Set

Dans un replica set, deux niveaux d'authentification sont nécessaires :

#### 1. Authentification Client → MongoDB

Configuration identique à une instance standalone.

#### 2. Authentification Interne (Inter-membres)

Les membres du replica set doivent s'authentifier entre eux pour la réplication.

**Méthodes** :

##### Option A : Keyfile (Simple)

```yaml
# mongod.conf sur TOUS les membres
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile
```

```bash
# Génération du keyfile (sur un membre, puis copier sur les autres)
openssl rand -base64 756 > /etc/mongodb/keyfile
chmod 400 /etc/mongodb/keyfile
chown mongodb:mongodb /etc/mongodb/keyfile

# Copier sur les autres membres
scp /etc/mongodb/keyfile mongo2:/etc/mongodb/
scp /etc/mongodb/keyfile mongo3:/etc/mongodb/
```

**⚠️ Limitations du keyfile** :
- Tous les membres partagent le même secret
- Compromission d'un membre = compromission de tous
- Rotation complexe (rolling restart)

##### Option B : x.509 Certificates (Recommandé en Production)

```yaml
# mongod.conf
security:
  authorization: enabled
  clusterAuthMode: x509

net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/member1.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    clusterFile: /etc/ssl/mongodb/member1-cluster.pem
```

**Avantages** :
- Certificats individuels par membre
- Révocation possible via CRL
- Conformité avec les standards PKI
- Pas de secret partagé

### Cluster Shardé

Un cluster shardé nécessite plusieurs niveaux d'authentification :

```
┌─────────────────────────────────────────────────────────────┐
│  CLIENTS                                                    │
│  ↓ Authentification utilisateur                             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  MONGOS (Query Routers)                                     │
│  ↓ Authentification interne                                 │
└─────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│  CONFIG SERVERS         │   │  SHARDS                 │
│  (Replica Set)          │   │  (Replica Sets)         │
│  ↓ Auth interne RS      │   │  ↓ Auth interne RS      │
└─────────────────────────┘   └─────────────────────────┘
```

**Configuration** :

```yaml
# Tous les composants (mongos, config servers, shards)
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile  # LE MÊME sur tous les composants

# OU avec x.509
security:
  authorization: enabled
  clusterAuthMode: x509
```

**⚠️ IMPORTANT** : Dans un cluster shardé, le keyfile doit être IDENTIQUE sur :
- Tous les mongos
- Tous les config servers
- Tous les shards (tous les membres de tous les replica sets)

## Authentification Interne vs Externe

### Authentification Interne (SCRAM, x.509)

**Caractéristiques** :
- Credentials stockés dans MongoDB (`admin.system.users`)
- Gestion des utilisateurs via MongoDB
- Pas de dépendance externe
- Idéal pour applications simples

**Workflow** :

```
Application → MongoDB
     |           |
     |           └── Vérifie credentials dans admin.system.users
     |           └── Accorde accès
     |
     └── Simple, autonome
```

### Authentification Externe (LDAP, Kerberos, OIDC)

**Caractéristiques** :
- Credentials gérés par un système externe
- Utilisateurs définis dans MongoDB avec `authSource: $external`
- Dépendance sur infrastructure externe
- Idéal pour intégration enterprise

**Workflow** :

```
Application → MongoDB → Système Externe (LDAP/Kerberos/IdP)
     |           |              |
     |           └── Forward ───┘
     |           ┌── Validation ─┘
     |           |
     |           └── Cherche utilisateur dans admin.system.users
     |           └── Applique rôles MongoDB
     |           └── Accorde accès
     |
     └── SSO, gestion centralisée
```

**Création d'utilisateur externe** :

```javascript
// LDAP
db.getSiblingDB("$external").createUser({
  user: "ldapuser@company.com",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// Kerberos
db.getSiblingDB("$external").createUser({
  user: "user@REALM.COM",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// x.509
db.getSiblingDB("$external").createUser({
  user: "CN=client,OU=IT,O=Company,L=Paris,C=FR",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})
```

## Choix du Mécanisme d'Authentification

### Arbre de Décision

```
┌─────────────────────────────────────┐
│  Besoin d'authentification ?        │
└─────────────────────────────────────┘
              │ OUI
              ▼
┌─────────────────────────────────────┐
│  MongoDB Enterprise disponible ?    │
└─────────────────────────────────────┘
       │ NON              │ OUI
       ▼                  ▼
┌────────────────┐   ┌─────────────────────────────┐
│  Community     │   │  Intégration AD/LDAP ?      │
│                │   └─────────────────────────────┘
│  Choix:        │          │ OUI          │ NON
│  • SCRAM       │          ▼              ▼
│  • x.509       │   ┌────────────┐  ┌──────────────┐
└────────────────┘   │   LDAP     │  │  Kerberos ?  │
                     └────────────┘  └──────────────┘
                                            │ NON
                                            ▼
                                    ┌────────────────┐
                                    │  SSO moderne ? │
                                    └────────────────┘
                                            │ OUI
                                            ▼
                                    ┌────────────────┐
                                    │     OIDC       │
                                    └────────────────┘
```

### Recommandations par Scénario

| Scénario | Mécanisme Recommandé | Justification |
|----------|---------------------|---------------|
| **Application standard** | SCRAM-SHA-256 | Simple, pas de dépendance externe, Community edition |
| **Haute sécurité** | x.509 | Pas de mots de passe, authentification mutuelle, révocation |
| **Entreprise avec AD** | LDAP | SSO, gestion centralisée, pas de duplication users |
| **Legacy enterprise** | Kerberos | Standard dans environnements Windows/Unix legacy |
| **Cloud-native / SaaS** | OIDC | Intégration moderne, compatibilité IdP cloud |
| **Machine-to-Machine** | x.509 | Automatisation, pas de secrets en clair, PKI |
| **Environnement mixte** | SCRAM + x.509 | Flexibilité, transition progressive |

### Considérations pour le Choix

**Privilégier SCRAM si** :
- ✅ MongoDB Community (pas besoin d'Enterprise)
- ✅ Infrastructure simple sans LDAP/AD
- ✅ Équipe réduite
- ✅ Budget limité
- ✅ Applications standard

**Privilégier x.509 si** :
- ✅ Exigences de sécurité élevées
- ✅ PKI déjà en place
- ✅ Pas de gestion de mots de passe souhaitée
- ✅ Authentification M2M
- ✅ Conformité stricte

**Privilégier LDAP si** :
- ✅ Active Directory déjà utilisé
- ✅ Nombreux utilisateurs à gérer
- ✅ SSO requis
- ✅ Gestion centralisée nécessaire
- ✅ Budget pour Enterprise

**Privilégier Kerberos si** :
- ✅ Infrastructure Kerberos existante
- ✅ Environnement hautement sécurisé
- ✅ Intégration avec systèmes legacy
- ✅ SSO requis sur Unix/Linux

**Privilégier OIDC si** :
- ✅ MongoDB Atlas ou Enterprise 7.0+
- ✅ Environnement cloud-native
- ✅ IdP moderne (Okta, Auth0, Azure AD)
- ✅ Applications web/mobile
- ✅ SSO moderne requis

## Migration entre Mécanismes

### Stratégie de Migration

MongoDB supporte **plusieurs mécanismes simultanément**, permettant une migration progressive.

#### Exemple : SCRAM → x.509

**Phase 1 : Préparation**

```bash
# 1. Configurer TLS sur tous les nœuds
# mongod.conf
net:
  tls:
    mode: preferTLS  # Accepte TLS et non-TLS temporairement
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
```

**Phase 2 : Authentification Hybride**

```yaml
# mongod.conf
security:
  authorization: enabled
  clusterAuthMode: keyFile  # Temporairement

net:
  tls:
    mode: preferTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowConnectionsWithoutCertificates: true  # Temporaire
```

**Phase 3 : Créer Utilisateurs x.509**

```javascript
// Créer les utilisateurs x.509 (sans supprimer les SCRAM)
db.getSiblingDB("$external").createUser({
  user: "CN=app01,OU=IT,O=Company",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})
```

**Phase 4 : Migration Progressive des Clients**

```javascript
// Client par client, passer de SCRAM à x.509
// Ancien (SCRAM):
mongodb://user:pass@host/db?authSource=admin

// Nouveau (x.509):
mongodb://CN=app01,OU=IT,O=Company@host/db?authMechanism=MONGODB-X509&authSource=$external&tls=true&tlsCertificateKeyFile=/path/to/client.pem
```

**Phase 5 : Durcissement**

Une fois tous les clients migrés :

```yaml
# mongod.conf
security:
  authorization: enabled
  clusterAuthMode: x509  # Passage à x.509 pour l'auth interne

net:
  tls:
    mode: requireTLS  # TLS obligatoire
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowConnectionsWithoutCertificates: false
```

**Phase 6 : Nettoyage**

```javascript
// Supprimer les anciens utilisateurs SCRAM
db.dropUser("olduser")
```

## Troubleshooting de l'Authentification

### Problèmes Courants

#### 1. "Authentication Failed"

**Symptômes** :
```
MongoServerError: Authentication failed.
```

**Causes possibles** :
- ✗ Mauvais username/password
- ✗ Mauvais `authSource`
- ✗ Utilisateur n'existe pas
- ✗ Mécanisme non supporté

**Diagnostic** :

```javascript
// Vérifier l'existence de l'utilisateur
use admin
db.system.users.find({ user: "username" }).pretty()

// Vérifier les mécanismes configurés
db.system.users.find({ user: "username" }, { mechanisms: 1 })

// Vérifier les logs
// Dans les logs MongoDB, chercher:
// "Authentication failed" avec le détail de l'erreur
```

**Solutions** :
```bash
# Tester avec authSource explicite
mongosh "mongodb://user:pass@host/db?authSource=admin"

# Réinitialiser le mot de passe
use admin
db.updateUser("username", { pwd: "newpassword" })
```

#### 2. "Not Authorized"

**Symptômes** :
```
MongoServerError: not authorized on database to execute command
```

**Cause** :
- ✓ Authentification réussie
- ✗ Mais manque de privilèges pour l'opération

**Solution** :
```javascript
// Vérifier les rôles de l'utilisateur
db.getUser("username")

// Accorder des rôles supplémentaires
db.grantRolesToUser("username", [
  { role: "readWrite", db: "targetdb" }
])
```

#### 3. Problèmes de Keyfile

**Symptômes** :
```
Error: couldn't connect to server, connection attempt failed:
HostUnreachable: unable to authenticate using mechanism SCRAM-SHA-256
```

**Causes** :
- ✗ Keyfile différent sur les membres
- ✗ Permissions incorrectes (doit être 400)
- ✗ Owner incorrect (doit être mongodb:mongodb)

**Solutions** :
```bash
# Vérifier les permissions
ls -la /etc/mongodb/keyfile
# Devrait afficher: -r-------- 1 mongodb mongodb

# Corriger les permissions
chmod 400 /etc/mongodb/keyfile
chown mongodb:mongodb /etc/mongodb/keyfile

# Vérifier que le keyfile est identique partout
md5sum /etc/mongodb/keyfile  # Sur chaque membre
```

#### 4. Problèmes TLS/x.509

**Symptômes** :
```
Error: SSL peer certificate validation failed
```

**Solutions** :
```bash
# Vérifier la validité du certificat
openssl x509 -in /etc/ssl/mongodb/server.pem -text -noout

# Vérifier la chaîne de certification
openssl verify -CAfile /etc/ssl/mongodb/ca.pem /etc/ssl/mongodb/server.pem

# Vérifier le CN vs hostname
openssl x509 -in /etc/ssl/mongodb/server.pem -noout -subject
# Doit correspondre au hostname utilisé pour la connexion
```

### Debug : Augmenter la Verbosité

```javascript
// Activer le debug pour l'authentification
db.setLogLevel(2, "accessControl")

// Via mongod.conf
systemLog:
  verbosity: 0
  component:
    accessControl:
      verbosity: 2
```

## Bonnes Pratiques de Production

### 1. Sécurité des Credentials

```javascript
// ❌ MAUVAIS : Password hardcodé
const uri = "mongodb://admin:admin123@localhost:27017/mydb";

// ✅ BON : Variable d'environnement
const uri = `mongodb://${process.env.MONGO_USER}:${process.env.MONGO_PASS}@localhost:27017/mydb`;

// ✅ MEILLEUR : Secret manager
const credentials = await getFromVault("mongodb/prod");
const uri = `mongodb://${credentials.username}:${credentials.password}@...`;
```

### 2. Rotation Régulière

```javascript
// Script de rotation automatisée
const rotatePassword = async (username) => {
  const newPassword = generateSecurePassword();

  // Mise à jour dans MongoDB
  db.getSiblingDB("admin").updateUser(username, {
    pwd: newPassword
  });

  // Stockage sécurisé
  await vault.write(`secret/mongodb/${username}`, {
    password: newPassword,
    rotatedAt: new Date().toISOString()
  });

  // Notification
  await notifyTeam(`Password rotated for ${username}`);
};

// Rotation tous les 90 jours
```

### 3. Restrictions d'Authentification

```javascript
// Limiter les sources de connexion autorisées
db.createUser({
  user: "webapp",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "myapp" }],
  authenticationRestrictions: [
    {
      // N'autorise QUE depuis le réseau applicatif
      clientSource: ["10.0.2.0/24"],
      // N'autorise QUE vers les instances MongoDB spécifiques
      serverAddress: ["10.0.1.100", "10.0.1.101", "10.0.1.102"]
    }
  ]
})
```

### 4. Monitoring de l'Authentification

```javascript
// Surveiller les échecs d'authentification
db.adminCommand({
  getLog: "global"
}).log.filter(entry =>
  entry.includes("Authentication failed")
).forEach(print);

// Avec audit logs
db.audit.find({
  atype: "authenticate",
  result: { $ne: 0 }  // 0 = succès, autre = échec
}).sort({ ts: -1 }).limit(100);

// Alertes sur tentatives multiples
// Détecter bruteforce: >5 échecs en 1 minute
```

### 5. Utiliser des Comptes Dédiés

```javascript
// ❌ MAUVAIS : Utiliser 'root' pour les applications
db.createUser({
  user: "myapp",
  pwd: "pass",
  roles: ["root"]
});

// ✅ BON : Comptes avec privilèges minimaux
db.createUser({
  user: "myapp_reader",
  pwd: passwordPrompt(),
  roles: [
    { role: "read", db: "myapp" }
  ]
});

db.createUser({
  user: "myapp_writer",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
});

db.createUser({
  user: "myapp_admin",
  pwd: passwordPrompt(),
  roles: [
    { role: "dbAdmin", db: "myapp" },
    { role: "readWrite", db: "myapp" }
  ]
});
```

### 6. Documentation des Utilisateurs

Maintenir une documentation des comptes :

```yaml
# users-inventory.yaml
users:
  - username: myapp_prod
    database: admin
    mechanism: SCRAM-SHA-256
    roles:
      - role: readWrite
        db: myapp_prod
    purpose: "Application principale - environnement production"
    owner: "Équipe Backend"
    created: "2024-01-15"
    lastRotation: "2024-11-01"

  - username: backup_service
    database: admin
    mechanism: SCRAM-SHA-256
    roles:
      - role: backup
      - role: clusterMonitor
    purpose: "Service de backup automatisé"
    owner: "Équipe DevOps"
    created: "2024-01-15"
    lastRotation: "2024-10-15"
```

## Conclusion

L'authentification est la fondation de la sécurité MongoDB. Les points essentiels :

1. **Toujours activer l'authentification** en production
2. **Choisir le mécanisme adapté** à votre infrastructure
3. **Utiliser des credentials robustes** et les gérer de manière sécurisée
4. **Appliquer le principe du moindre privilège**
5. **Automatiser la rotation** des credentials
6. **Monitorer les tentatives d'authentification**
7. **Documenter** les comptes et leur usage

Les sections suivantes détaillent chaque mécanisme d'authentification avec des configurations concrètes et des exemples d'implémentation.

---

**Prochaines Sections** :
- **11.2.1** : SCRAM (défaut) - Configuration détaillée
- **11.2.2** : x.509 - Authentification par certificats
- **11.2.3** : LDAP - Intégration Active Directory
- **11.2.4** : Kerberos - Authentification enterprise

⏭️ [SCRAM (défaut)](/11-securite/02.1-scram.md)
