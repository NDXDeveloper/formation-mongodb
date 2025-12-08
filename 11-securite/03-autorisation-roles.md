🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3 Autorisation et Rôles

## Introduction

L'autorisation dans MongoDB détermine ce qu'un utilisateur authentifié est autorisé à faire. Alors que l'authentification vérifie **qui** vous êtes, l'autorisation contrôle **ce que** vous pouvez faire. MongoDB utilise un système de contrôle d'accès basé sur les rôles (RBAC - Role-Based Access Control) qui permet une gestion granulaire et flexible des permissions.

### Authentification vs Autorisation

```
┌─────────────────────────────────────────────────────────────────┐
│                      AUTHENTIFICATION                           │
│  Question : "Qui êtes-vous ?"                                   │
│  Mécanismes : SCRAM, x.509, LDAP, Kerberos                      │
│  Résultat : Identité vérifiée                                   │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                       AUTORISATION                              │
│  Question : "Que pouvez-vous faire ?"                           │
│  Mécanisme : RBAC (Role-Based Access Control)                   │
│  Résultat : Permissions accordées                               │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                     ACCÈS AUX DONNÉES                           │
│  Actions : find(), insert(), update(), delete(), etc.           │
│  Resources : Databases, Collections, Cluster                    │
└─────────────────────────────────────────────────────────────────┘
```

**Exemple concret** :

```javascript
// Authentification : John Doe prouve son identité
// Via SCRAM, x.509, LDAP, ou Kerberos
mongosh -u john.doe -p --authenticationDatabase admin

// Autorisation : Quelles actions John peut effectuer
// Défini par ses rôles : readWrite sur "production", read sur "analytics"
use production
db.orders.find()      // ✅ Autorisé (role: readWrite)
db.orders.insertOne() // ✅ Autorisé (role: readWrite)

use analytics
db.metrics.find()     // ✅ Autorisé (role: read)
db.metrics.insertOne() // ❌ Refusé (role: read, pas write)

use admin
db.createUser()       // ❌ Refusé (pas de role userAdmin)
```

## Architecture RBAC de MongoDB

### Composants du Système RBAC

```
┌───────────────────────────────────────────────────────────────────┐
│                            USER                                   │
│  • Identité unique (username)                                     │
│  • Base de données d'authentification (authSource)                │
│  • Assigné à un ou plusieurs ROLES                                │
└───────────────────────────────────────────────────────────────────┘
                            │
                            │ has
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                            ROLE                                   │
│  • Ensemble de PRIVILEGES                                         │
│  • Peut hériter d'autres ROLES                                    │
│  • Défini au niveau database                                      │
└───────────────────────────────────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
┌───────────────────────┐    ┌──────────────────────────┐
│     PRIVILEGES        │    │    INHERITED ROLES       │
│  • Actions            │    │  • readWrite             │
│  • Resources          │    │  • dbAdmin               │
└───────────────────────┘    └──────────────────────────┘
        │
        ├─── ACTIONS (find, insert, update, delete, createIndex, ...)
        │
        └─── RESOURCES (database, collection, cluster)
```

### Hiérarchie Conceptuelle

```
CLUSTER (Niveau système)
    │
    ├── DATABASE 1
    │   ├── Collection A
    │   ├── Collection B
    │   └── Collection C
    │
    ├── DATABASE 2
    │   ├── Collection D
    │   └── Collection E
    │
    └── DATABASE 3
        └── Collection F

Permissions peuvent être définies à chaque niveau :
- CLUSTER : Actions sur le système entier (shutdown, replSetGetStatus)
- DATABASE : Actions sur toutes les collections d'une base
- COLLECTION : Actions sur une collection spécifique
```

## Concepts Fondamentaux

### 1. Rôles (Roles)

Un rôle est une collection de privilèges qui définissent les actions qu'un utilisateur peut effectuer.

**Types de rôles** :
- **Built-in roles** : Rôles prédéfinis par MongoDB
- **User-defined roles** : Rôles personnalisés créés par les administrateurs

**Caractéristiques** :
- Définis dans une base de données spécifique
- Peuvent hériter d'autres rôles
- Peuvent être assignés à plusieurs utilisateurs
- Peuvent être modifiés dynamiquement

**Structure d'un rôle** :

```javascript
{
  _id: "myDatabase.myRole",
  role: "myRole",
  db: "myDatabase",

  // Privilèges directs
  privileges: [
    {
      resource: { db: "myDatabase", collection: "myCollection" },
      actions: ["find", "insert", "update"]
    }
  ],

  // Rôles hérités
  roles: [
    { role: "read", db: "otherDatabase" }
  ]
}
```

### 2. Privilèges (Privileges)

Un privilège est une combinaison d'une **resource** et d'**actions** autorisées sur cette resource.

```javascript
{
  resource: {
    db: "production",
    collection: "orders"
  },
  actions: [
    "find",
    "insert",
    "update",
    "remove"
  ]
}
```

**Composants** :
- **Resource** : Objet sur lequel s'appliquent les actions
- **Actions** : Opérations autorisées

### 3. Resources

Les resources définissent la portée des permissions.

#### Types de Resources

**A. Collection spécifique** :

```javascript
{ db: "production", collection: "orders" }
```

**B. Toutes les collections d'une base** :

```javascript
{ db: "production", collection: "" }
```

**C. Collection spécifique dans toutes les bases** :

```javascript
{ db: "", collection: "logs" }
```

**D. Toutes les collections de toutes les bases** :

```javascript
{ db: "", collection: "" }
```

**E. Cluster (système entier)** :

```javascript
{ cluster: true }
```

**F. Base de données (métadonnées)** :

```javascript
{ db: "production", collection: "system.indexes" }
```

#### Hiérarchie des Resources

```
{ cluster: true }
    ↓ Plus large
{ db: "", collection: "" }
    ↓
{ db: "production", collection: "" }
    ↓
{ db: "production", collection: "orders" }
    ↓ Plus spécifique
```

**Principe** : Plus la resource est spécifique, plus le contrôle est granulaire.

### 4. Actions

Les actions sont les opérations spécifiques qu'un utilisateur peut effectuer sur une resource.

#### Catégories d'Actions

**A. Actions de Lecture** :
```
find, listCollections, listDatabases, listIndexes
```

**B. Actions d'Écriture** :
```
insert, update, remove, createCollection, dropCollection
```

**C. Actions d'Administration de Base** :
```
createIndex, dropIndex, collStats, dbStats, enableSharding
```

**D. Actions d'Administration Utilisateur** :
```
createUser, dropUser, grantRole, revokeRole, viewUser
```

**E. Actions de Cluster** :
```
serverStatus, replSetGetStatus, shardingState, shutdown
```

**F. Actions Internes** :
```
internal (réservées au système)
```

#### Actions Courantes

| Action | Description | Niveau |
|--------|-------------|--------|
| `find` | Requêtes de lecture | Collection |
| `insert` | Insertion de documents | Collection |
| `update` | Modification de documents | Collection |
| `remove` | Suppression de documents | Collection |
| `createCollection` | Créer une collection | Database |
| `dropCollection` | Supprimer une collection | Database |
| `createIndex` | Créer un index | Collection |
| `dropIndex` | Supprimer un index | Collection |
| `createUser` | Créer un utilisateur | Database |
| `dropUser` | Supprimer un utilisateur | Database |
| `grantRole` | Accorder un rôle | Database |
| `revokeRole` | Révoquer un rôle | Database |
| `serverStatus` | Voir le statut du serveur | Cluster |
| `shutdown` | Arrêter le serveur | Cluster |

**Liste complète** : https://docs.mongodb.com/manual/reference/privilege-actions/

### 5. Héritage de Rôles

Les rôles peuvent hériter d'autres rôles, créant une hiérarchie de permissions.

```javascript
// Rôle de base
db.createRole({
  role: "dataEntry",
  privileges: [
    {
      resource: { db: "production", collection: "orders" },
      actions: ["find", "insert"]
    }
  ],
  roles: []
})

// Rôle qui hérite
db.createRole({
  role: "seniorDataEntry",
  privileges: [
    {
      resource: { db: "production", collection: "orders" },
      actions: ["update", "remove"]  // Privilèges additionnels
    }
  ],
  roles: [
    { role: "dataEntry", db: "production" }  // Hérite de dataEntry
  ]
})

// Résultat : seniorDataEntry a find, insert, update, remove
```

**Avantages de l'héritage** :
- Réduction de la duplication
- Hiérarchies de permissions claires
- Maintenance simplifiée
- Évolution facile des rôles

**Représentation graphique** :

```
        root
          │
    ┌─────┴─────┐
    ▼           ▼
dbOwner    userAdminAnyDatabase
    │
    ├─── readWrite
    │       │
    │       ├─── read
    │       └─── (write privileges)
    │
    └─── dbAdmin
            │
            └─── (admin privileges)
```

## Principe du Moindre Privilège

Le principe fondamental de sécurité : **accorder uniquement les permissions strictement nécessaires**.

### Application Pratique

**❌ Mauvais** :

```javascript
// Trop de permissions
db.createUser({
  user: "webapp",
  pwd: "password",
  roles: [
    { role: "root", db: "admin" }  // Accès total au système!
  ]
})
```

**✅ Bon** :

```javascript
// Permissions minimales nécessaires
db.createUser({
  user: "webapp",
  pwd: "password",
  roles: [
    { role: "readWrite", db: "production" }  // Uniquement la base nécessaire
  ]
})
```

### Stratégie par Persona

#### Développeur

```javascript
db.createUser({
  user: "developer_john",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "development" },
    { role: "dbAdmin", db: "development" }  // Créer index, voir stats
  ]
})
```

#### Application Web

```javascript
db.createUser({
  user: "webapp_prod",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "production" }
  ]
})
```

#### Analyste de Données

```javascript
db.createUser({
  user: "analyst_jane",
  pwd: passwordPrompt(),
  roles: [
    { role: "read", db: "production" },
    { role: "readWrite", db: "analytics" }
  ]
})
```

#### Service de Backup

```javascript
db.createUser({
  user: "backup_service",
  pwd: passwordPrompt(),
  roles: [
    { role: "backup", db: "admin" },
    { role: "clusterMonitor", db: "admin" }
  ]
})
```

#### DBA (Database Administrator)

```javascript
db.createUser({
  user: "dba_admin",
  pwd: passwordPrompt(),
  roles: [
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" }
  ]
})
```

## Structure des Rôles dans MongoDB

### Base de Données admin

La base `admin` a un statut spécial :

**Privilèges spéciaux** :
- Rôles définis dans `admin` peuvent avoir des permissions sur tout le cluster
- Suffixes `AnyDatabase` pour permissions globales
- Seule base où certains rôles cluster sont disponibles

```javascript
// Rôle dans admin avec portée globale
use admin
db.createRole({
  role: "globalReader",
  privileges: [
    {
      resource: { db: "", collection: "" },  // Toutes les bases
      actions: ["find", "listCollections", "listDatabases"]
    }
  ],
  roles: []
})
```

### Bases de Données Utilisateur

Rôles spécifiques à une base de données :

```javascript
// Rôle spécifique à la base "production"
use production
db.createRole({
  role: "orderManager",
  privileges: [
    {
      resource: { db: "production", collection: "orders" },
      actions: ["find", "insert", "update", "remove"]
    },
    {
      resource: { db: "production", collection: "customers" },
      actions: ["find"]
    }
  ],
  roles: []
})
```

### Rôles Cross-Database

Un rôle peut donner des permissions sur plusieurs bases :

```javascript
use admin
db.createRole({
  role: "multiDbUser",
  privileges: [],
  roles: [
    { role: "readWrite", db: "production" },
    { role: "read", db: "analytics" },
    { role: "read", db: "logs" }
  ]
})
```

## Gestion des Rôles

### Création de Rôles

```javascript
// Rôle simple
use myDatabase
db.createRole({
  role: "myRole",
  privileges: [
    {
      resource: { db: "myDatabase", collection: "myCollection" },
      actions: ["find", "insert", "update"]
    }
  ],
  roles: []
})

// Rôle avec héritage
db.createRole({
  role: "seniorRole",
  privileges: [
    {
      resource: { db: "myDatabase", collection: "sensitiveData" },
      actions: ["find"]
    }
  ],
  roles: [
    { role: "myRole", db: "myDatabase" }  // Hérite de myRole
  ]
})
```

### Modification de Rôles

#### Ajouter des Privilèges

```javascript
db.grantPrivilegesToRole("myRole", [
  {
    resource: { db: "myDatabase", collection: "newCollection" },
    actions: ["find", "insert"]
  }
])
```

#### Retirer des Privilèges

```javascript
db.revokePrivilegesFromRole("myRole", [
  {
    resource: { db: "myDatabase", collection: "oldCollection" },
    actions: ["remove"]
  }
])
```

#### Ajouter des Rôles Hérités

```javascript
db.grantRolesToRole("seniorRole", [
  { role: "read", db: "analytics" }
])
```

#### Retirer des Rôles Hérités

```javascript
db.revokeRolesFromRole("seniorRole", [
  { role: "read", db: "analytics" }
])
```

### Inspection des Rôles

```javascript
// Voir les détails d'un rôle
use myDatabase
db.getRole("myRole", { showPrivileges: true })

// Résultat :
{
  "_id" : "myDatabase.myRole",
  "role" : "myRole",
  "db" : "myDatabase",
  "privileges" : [
    {
      "resource" : { "db" : "myDatabase", "collection" : "myCollection" },
      "actions" : [ "find", "insert", "update" ]
    }
  ],
  "roles" : [ ]
}

// Voir tous les rôles d'une base
db.getRoles({ showPrivileges: true, showBuiltinRoles: true })

// Voir les rôles avec héritage complet
db.getRole("myRole", { showPrivileges: true, showAuthenticationRestrictions: true })
```

### Suppression de Rôles

```javascript
use myDatabase
db.dropRole("myRole")

// Vérifier
db.getRoles()
```

## Gestion des Utilisateurs et Rôles

### Création d'Utilisateurs avec Rôles

```javascript
use admin
db.createUser({
  user: "appUser",
  pwd: passwordPrompt(),

  // Multiples rôles possibles
  roles: [
    { role: "readWrite", db: "production" },
    { role: "read", db: "analytics" },
    { role: "clusterMonitor", db: "admin" }
  ]
})
```

### Modification des Rôles d'un Utilisateur

#### Accorder des Rôles

```javascript
use admin
db.grantRolesToUser("appUser", [
  { role: "dbAdmin", db: "production" }
])
```

#### Révoquer des Rôles

```javascript
db.revokeRolesFromUser("appUser", [
  { role: "clusterMonitor", db: "admin" }
])
```

### Inspection des Permissions Utilisateur

```javascript
// Voir les rôles d'un utilisateur
use admin
db.getUser("appUser")

// Résultat :
{
  "_id" : "admin.appUser",
  "user" : "appUser",
  "db" : "admin",
  "roles" : [
    { "role" : "readWrite", "db" : "production" },
    { "role" : "read", "db" : "analytics" },
    { "role" : "dbAdmin", "db" : "production" }
  ]
}

// Voir avec les privilèges effectifs
db.getUser("appUser", { showPrivileges: true })
```

### Vérification des Permissions

```javascript
// Vérifier si un utilisateur a une permission spécifique
use admin
db.runCommand({
  usersInfo: { user: "appUser", db: "admin" },
  showPrivileges: true,
  showAuthenticationRestrictions: true
})
```

## Scopes et Contextes

### Database Scope

Les rôles sont définis dans le contexte d'une base de données :

```javascript
// Créer un rôle dans "production"
use production
db.createRole({
  role: "productionRole",
  privileges: [...],
  roles: []
})

// Ce rôle est accessible comme:
{ role: "productionRole", db: "production" }
```

### Collection Scope

Privilèges peuvent cibler des collections spécifiques :

```javascript
{
  resource: { db: "production", collection: "orders" },
  actions: ["find", "insert"]
}

// vs toutes les collections
{
  resource: { db: "production", collection: "" },
  actions: ["find"]
}
```

### Cluster Scope

Actions au niveau du cluster entier :

```javascript
{
  resource: { cluster: true },
  actions: ["serverStatus", "replSetGetStatus"]
}
```

## Évaluation des Permissions

MongoDB évalue les permissions de la manière suivante :

```
1. Authentification réussie
   ↓
2. Récupération des rôles de l'utilisateur
   ↓
3. Expansion récursive des rôles hérités
   ↓
4. Agrégation de tous les privilèges
   ↓
5. Vérification si l'action demandée est autorisée
   ↓
6. Accès accordé ou refusé
```

**Exemple d'évaluation** :

```javascript
// Utilisateur avec rôles
User: john.doe
Roles:
  - readWrite@production
  - read@analytics

// Tentative d'action
db.production.orders.find()

// Évaluation :
1. john.doe est authentifié ✓
2. Rôles : readWrite@production, read@analytics
3. readWrite inclut : read + write privileges
4. find() est une action de lecture
5. Resource: production.orders
6. readWrite@production donne accès à toutes collections de production
7. ✅ Accès accordé
```

## Bonnes Pratiques de Production

### 1. Principe du Moindre Privilège (Rappel)

```javascript
// ❌ À ÉVITER
roles: [{ role: "root", db: "admin" }]

// ✅ RECOMMANDÉ
roles: [
  { role: "readWrite", db: "myapp" }
]
```

### 2. Utiliser des Rôles Personnalisés pour Logique Métier

```javascript
// Créer un rôle métier spécifique
use production
db.createRole({
  role: "customerServiceAgent",
  privileges: [
    {
      resource: { db: "production", collection: "customers" },
      actions: ["find", "update"]  // Peut consulter et mettre à jour
    },
    {
      resource: { db: "production", collection: "orders" },
      actions: ["find"]  // Lecture seule des commandes
    }
  ],
  roles: []
})
```

### 3. Séparer les Comptes par Fonction

```javascript
// Compte pour l'application web
db.createUser({
  user: "webapp_prod",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "production" }]
})

// Compte pour les tâches batch
db.createUser({
  user: "batch_processor",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "production" },
    { role: "dbAdmin", db: "production" }  // Pour créer des index
  ]
})

// Compte pour la surveillance
db.createUser({
  user: "monitoring_agent",
  pwd: passwordPrompt(),
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "production" }
  ]
})
```

### 4. Documenter les Rôles Personnalisés

```javascript
// Inclure une description dans les métadonnées
use production
db.system.roles.updateOne(
  { _id: "production.customerServiceAgent" },
  {
    $set: {
      description: "Rôle pour agents service client : consultation clients et commandes, modification clients",
      createdBy: "DBA Team",
      createdAt: new Date("2024-12-08"),
      owner: "Customer Service Department"
    }
  }
)

// Maintenir une documentation externe
```

### 5. Révision Périodique des Permissions

```javascript
// Script d'audit mensuel
function auditUserRoles() {
  use admin

  const users = db.getUsers();
  const report = [];

  users.forEach(user => {
    const userInfo = {
      username: user.user,
      db: user.db,
      roles: user.roles,
      lastModified: user.lastModified || "unknown"
    };

    // Vérifier si root ou permissions trop larges
    const hasRoot = user.roles.some(r => r.role === 'root');
    const hasAnyDatabase = user.roles.some(r => r.role.includes('AnyDatabase'));

    if (hasRoot || hasAnyDatabase) {
      userInfo.warning = "Permissions très larges - revue requise";
    }

    report.push(userInfo);
  });

  return report;
}

// Exécuter et analyser
const auditReport = auditUserRoles();
printjson(auditReport);
```

### 6. Utiliser authenticationRestrictions

```javascript
// Limiter l'accès par IP et serveur
db.createUser({
  user: "webapp_prod",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "production" }],

  authenticationRestrictions: [
    {
      // Uniquement depuis le réseau applicatif
      clientSource: ["10.0.2.0/24"],

      // Uniquement vers les serveurs MongoDB spécifiques
      serverAddress: ["10.0.1.100", "10.0.1.101", "10.0.1.102"]
    }
  ]
})
```

### 7. Séparer Lecture et Écriture

```javascript
// Compte lecture seule pour analytics
db.createUser({
  user: "analytics_reader",
  pwd: passwordPrompt(),
  roles: [{ role: "read", db: "production" }]
})

// Compte avec écriture pour l'application
db.createUser({
  user: "app_writer",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "production" }]
})
```

### 8. Isoler les Données Sensibles

```javascript
// Collection sensible avec permissions restrictives
use production
db.createRole({
  role: "piiAccess",
  privileges: [
    {
      resource: { db: "production", collection: "customer_pii" },
      actions: ["find", "update"]
    }
  ],
  roles: []
})

// Attribuer uniquement aux personnes autorisées
db.createUser({
  user: "compliance_officer",
  pwd: passwordPrompt(),
  roles: [
    { role: "piiAccess", db: "production" },
    { role: "read", db: "production" }  // Lecture générale
  ]
})
```

## Patterns Courants

### Pattern 1 : Environnements Multiples

```javascript
// Développement
db.createUser({
  user: "dev_user",
  pwd: "dev123",
  roles: [
    { role: "readWrite", db: "myapp_dev" },
    { role: "dbAdmin", db: "myapp_dev" }
  ]
})

// Staging
db.createUser({
  user: "staging_user",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp_staging" }
  ]
})

// Production (permissions minimales)
db.createUser({
  user: "prod_user",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp_prod" }
  ]
})
```

### Pattern 2 : Séparation Lecture/Écriture

```javascript
// Utilisateur lecture (analytics, reporting)
db.createUser({
  user: "reader",
  pwd: passwordPrompt(),
  roles: [{ role: "read", db: "production" }]
})

// Utilisateur écriture (application)
db.createUser({
  user: "writer",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "production" }]
})

// Utilisateur admin (DBA)
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [
    { role: "dbOwner", db: "production" }
  ]
})
```

### Pattern 3 : Rôles par Équipe

```javascript
// Équipe Frontend
db.createRole({
  role: "frontendTeam",
  privileges: [
    {
      resource: { db: "production", collection: "users" },
      actions: ["find", "insert", "update"]
    },
    {
      resource: { db: "production", collection: "sessions" },
      actions: ["find", "insert", "update", "remove"]
    }
  ],
  roles: []
})

// Équipe Backend
db.createRole({
  role: "backendTeam",
  privileges: [],
  roles: [
    { role: "readWrite", db: "production" },
    { role: "dbAdmin", db: "production" }
  ]
})

// Équipe Data
db.createRole({
  role: "dataTeam",
  privileges: [],
  roles: [
    { role: "read", db: "production" },
    { role: "readWrite", db: "analytics" }
  ]
})
```

## Surveillance et Audit

### Monitoring des Accès

```javascript
// Vérifier qui a quelles permissions
use admin
db.system.users.find().forEach(user => {
  print(`User: ${user.user}@${user.db}`);
  print(`Roles: ${JSON.stringify(user.roles)}`);
  print("---");
})
```

### Audit Log (Enterprise)

```yaml
# /etc/mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log

  # Filtrer les événements de permissions
  filter: '{
    "atype": {
      "$in": [
        "authCheck",
        "createUser",
        "dropUser",
        "grantRolesToUser",
        "revokeRolesFromUser",
        "createRole",
        "dropRole",
        "grantPrivilegesToRole",
        "revokePrivilegesFromRole"
      ]
    }
  }'
```

### Détection d'Anomalies

```javascript
// Script pour détecter les accès inhabituels
use admin
db.system.auditlog.aggregate([
  {
    $match: {
      atype: "authCheck",
      result: { $ne: 0 }  // Échecs d'autorisation
    }
  },
  {
    $group: {
      _id: {
        user: "$param.user",
        ns: "$param.ns",
        command: "$param.command"
      },
      count: { $sum: 1 },
      lastAttempt: { $max: "$ts" }
    }
  },
  {
    $match: {
      count: { $gte: 10 }  // Plus de 10 échecs
    }
  },
  {
    $sort: { count: -1 }
  }
])
```

## Outils d'Administration

### Script de Gestion des Rôles

```javascript
// roles-manager.js

class RolesManager {
  constructor(db) {
    this.db = db;
  }

  // Créer un rôle standard
  createStandardRole(roleName, database, collections, permissions) {
    const privileges = collections.map(coll => ({
      resource: { db: database, collection: coll },
      actions: permissions
    }));

    this.db.getSiblingDB(database).createRole({
      role: roleName,
      privileges: privileges,
      roles: []
    });

    print(`✅ Role ${roleName} created in ${database}`);
  }

  // Cloner un rôle
  cloneRole(sourceRole, sourceDb, targetRole, targetDb) {
    const role = this.db.getSiblingDB(sourceDb).getRole(sourceRole, {
      showPrivileges: true
    });

    if (!role) {
      print(`❌ Source role not found: ${sourceRole}@${sourceDb}`);
      return;
    }

    this.db.getSiblingDB(targetDb).createRole({
      role: targetRole,
      privileges: role.privileges,
      roles: role.roles
    });

    print(`✅ Role cloned: ${sourceRole}@${sourceDb} → ${targetRole}@${targetDb}`);
  }

  // Comparer deux rôles
  compareRoles(role1, db1, role2, db2) {
    const r1 = this.db.getSiblingDB(db1).getRole(role1, { showPrivileges: true });
    const r2 = this.db.getSiblingDB(db2).getRole(role2, { showPrivileges: true });

    print(`\n=== Comparing ${role1}@${db1} vs ${role2}@${db2} ===`);
    print(`Privileges in ${role1}: ${r1.privileges.length}`);
    print(`Privileges in ${role2}: ${r2.privileges.length}`);
    print(`Inherited roles in ${role1}: ${r1.roles.length}`);
    print(`Inherited roles in ${role2}: ${r2.roles.length}`);
  }

  // Exporter un rôle
  exportRole(roleName, database) {
    const role = this.db.getSiblingDB(database).getRole(roleName, {
      showPrivileges: true
    });

    if (!role) {
      print(`❌ Role not found: ${roleName}@${database}`);
      return null;
    }

    const export_data = {
      role: role.role,
      db: role.db,
      privileges: role.privileges,
      roles: role.roles
    };

    print(JSON.stringify(export_data, null, 2));
    return export_data;
  }
}

// Usage
const manager = new RolesManager(db);
manager.createStandardRole("apiUser", "production", ["users", "sessions"], ["find", "insert", "update"]);
```

## Conclusion

L'autorisation basée sur les rôles (RBAC) dans MongoDB offre un contrôle granulaire et flexible des permissions. Les concepts clés à retenir :

**Principes fondamentaux** :
- ✅ **Authentification** vérifie l'identité, **Autorisation** contrôle l'accès
- ✅ **RBAC** permet une gestion centralisée et scalable des permissions
- ✅ **Moindre privilège** : accorder uniquement les permissions nécessaires
- ✅ **Rôles** combinant privileges et héritage pour réutilisabilité

**Composants RBAC** :
- **Users** : Identités avec rôles assignés
- **Roles** : Collections de privilèges (built-in ou custom)
- **Privileges** : Combinaisons de resources et actions
- **Resources** : Databases, collections, ou cluster
- **Actions** : Opérations spécifiques (find, insert, update, etc.)

**Bonnes pratiques** :
1. Utiliser le principe du moindre privilège systématiquement
2. Créer des rôles personnalisés pour la logique métier
3. Séparer les comptes par fonction (app, backup, monitoring)
4. Documenter tous les rôles personnalisés
5. Réviser les permissions périodiquement
6. Utiliser authenticationRestrictions (IP, serveur)
7. Séparer lecture et écriture quand possible
8. Isoler les données sensibles avec permissions restrictives
9. Activer l'audit pour traçabilité (Enterprise)
10. Monitorer les tentatives d'accès non autorisées

Les sections suivantes détaillent les rôles intégrés, les rôles personnalisés, et la gestion granulaire des privilèges pour une mise en œuvre complète de RBAC en production.

---

**Prochaines Sections** :
- **11.3.1** : Rôles intégrés - Catalogue complet des rôles built-in
- **11.3.2** : Rôles personnalisés - Création et gestion de rôles métier
- **11.3.3** : Privilèges et actions - Détail des permissions disponibles

⏭️ [Rôles intégrés](/11-securite/03.1-roles-integres.md)
