🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4 Gestion des Utilisateurs

## Introduction

La gestion des utilisateurs MongoDB est une tâche critique pour la sécurité et l'opérabilité d'un système en production. Elle englobe la création, la modification, la suppression et l'audit des comptes utilisateurs, ainsi que la gestion de leur cycle de vie complet. Une gestion rigoureuse des utilisateurs garantit que seules les personnes et services autorisés ont accès aux données, avec les permissions appropriées.

### Importance de la Gestion des Utilisateurs

**Sécurité** :
- Contrôle d'accès granulaire aux données
- Principe du moindre privilège
- Traçabilité des actions (qui a fait quoi)
- Révocation rapide en cas de compromission

**Opérationnalité** :
- Provisioning automatisé de nouveaux services
- Déprovisioning lors de départs ou changements
- Gestion des credentials sans intervention manuelle
- Scalabilité pour centaines/milliers d'utilisateurs

**Conformité** :
- Audit trail complet
- Respect des politiques de sécurité
- Rotation obligatoire des mots de passe
- Séparation des responsabilités

### Types d'Utilisateurs

```
┌────────────────────────────────────────────────────────────┐
│                    UTILISATEURS MONGODB                    │
└────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   HUMAINS   │    │  SERVICES   │    │  SYSTÈME    │
└─────────────┘    └─────────────┘    └─────────────┘
        │                   │                   │
        ├─ DBA              ├─ Applications     ├─ Monitoring
        ├─ Développeurs     ├─ Microservices    ├─ Backup
        ├─ Analystes        ├─ Batch jobs       ├─ Replication
        └─ Support          └─ APIs             └─ Internal
```

**Utilisateurs Humains** :
- Accès interactif (mongosh, compass)
- Authentification forte requise (MFA recommandé)
- Sessions limitées dans le temps
- Audit détaillé

**Comptes de Service** :
- Authentification automatisée (keytabs, certificats)
- Credentials stockés en coffre-fort (Vault)
- Permissions minimales spécifiques
- Rotation automatique

**Comptes Système** :
- Internes à MongoDB (replication, sharding)
- Gérés automatiquement
- Ne pas modifier manuellement

## Création d'Utilisateurs

### Syntaxe de Base

```javascript
use admin
db.createUser({
  user: "username",
  pwd: "password",  // ou passwordPrompt()
  roles: [
    { role: "roleName", db: "databaseName" }
  ]
})
```

### Création Interactive Sécurisée

```javascript
use admin
db.createUser({
  user: "john.doe",
  pwd: passwordPrompt(),  // Prompt interactif (recommandé)
  roles: [
    { role: "readWrite", db: "production" },
    { role: "read", db: "analytics" }
  ]
})

// Prompt affichera :
// Enter password:
// ***********
```

**Avantages** :
- Mot de passe non visible dans l'historique
- Pas de trace dans les logs
- Saisie masquée

### Création avec Métadonnées

```javascript
use admin
db.createUser({
  user: "webapp_prod_001",
  pwd: passwordPrompt(),

  roles: [
    { role: "readWrite", db: "production" }
  ],

  // Métadonnées personnalisées
  customData: {
    fullName: "Production Web Application",
    team: "Engineering",
    environment: "production",
    owner: "devops@company.com",
    createdDate: new Date(),
    expirationDate: new Date("2025-12-31"),
    purpose: "Main application database access",
    ticketNumber: "JIRA-12345"
  }
})
```

**Usage des métadonnées** :
- Documentation intégrée
- Audit et traçabilité
- Gestion automatisée (expiration)
- Identification du propriétaire

### Création Multi-Rôles

```javascript
use admin
db.createUser({
  user: "dba_admin",
  pwd: passwordPrompt(),

  roles: [
    // Lecture/écriture sur toutes les bases
    { role: "readWriteAnyDatabase", db: "admin" },

    // Administration de toutes les bases
    { role: "dbAdminAnyDatabase", db: "admin" },

    // Monitoring du cluster
    { role: "clusterMonitor", db: "admin" },

    // Rôle personnalisé
    { role: "customDbaRole", db: "admin" }
  ],

  customData: {
    fullName: "Senior DBA - John Smith",
    department: "Database Operations",
    manager: "jane.manager@company.com"
  }
})
```

### Création par Environnement

#### Développement

```javascript
use development
db.createUser({
  user: "dev_john",
  pwd: "Dev123!",  // Acceptable en dev
  roles: [
    { role: "dbOwner", db: "development" }  // Contrôle total en dev
  ],
  customData: {
    environment: "development",
    expiresAfter: 90  // jours
  }
})
```

#### Production

```javascript
use admin
db.createUser({
  user: "prod_webapp",
  pwd: passwordPrompt(),  // Toujours prompt en prod

  roles: [
    { role: "readWrite", db: "production" }  // Permissions minimales
  ],

  customData: {
    environment: "production",
    application: "web-application",
    approvedBy: "security-team",
    createdAt: new Date(),
    reviewDate: new Date("2025-06-01")  // Révision semestrielle
  },

  // Restrictions réseau (si supporté)
  authenticationRestrictions: [
    {
      clientSource: ["10.0.0.0/8"],
      serverAddress: ["192.168.1.100", "192.168.1.101"]
    }
  ]
})
```

### Création de Comptes de Service

```javascript
use admin
db.createUser({
  user: "svc_backup",
  pwd: passwordPrompt(),

  roles: [
    { role: "backup", db: "admin" },
    { role: "clusterMonitor", db: "admin" }
  ],

  customData: {
    accountType: "service",
    serviceName: "Automated Backup Service",
    owner: "ops-team@company.com",
    rotationSchedule: "quarterly",
    lastRotation: new Date(),
    nextRotation: new Date("2025-03-01")
  }
})
```

### Création en Masse (Bulk)

```javascript
// Script de création en masse
const users = [
  {
    user: "analyst_1",
    pwd: "TempPass123!",
    roles: [{ role: "read", db: "analytics" }],
    team: "Data Analytics"
  },
  {
    user: "analyst_2",
    pwd: "TempPass456!",
    roles: [{ role: "read", db: "analytics" }],
    team: "Data Analytics"
  },
  {
    user: "support_1",
    pwd: "TempPass789!",
    roles: [{ role: "customerSupportAgent", db: "production" }],
    team: "Customer Support"
  }
];

use admin
users.forEach(userData => {
  try {
    db.createUser({
      user: userData.user,
      pwd: userData.pwd,
      roles: userData.roles,
      customData: {
        team: userData.team,
        createdDate: new Date(),
        mustChangePassword: true  // Forcer changement au 1er login
      }
    });
    print(`✅ Created user: ${userData.user}`);
  } catch (e) {
    print(`❌ Error creating ${userData.user}: ${e}`);
  }
});
```

## Modification d'Utilisateurs

### Changement de Mot de Passe

#### Par l'Administrateur

```javascript
use admin
db.changeUserPassword("username", passwordPrompt())

// Ou avec mot de passe direct (non recommandé)
db.changeUserPassword("username", "newPassword123!")
```

#### Par l'Utilisateur Lui-Même

```javascript
// L'utilisateur doit avoir la permission changeOwnPassword
use admin
db.updateUser("myusername", {
  pwd: passwordPrompt()
})
```

#### Rotation Automatique avec Notification

```javascript
function rotateUserPassword(username, notifyEmail) {
  // Générer un mot de passe fort
  const newPassword = generateSecurePassword(32);

  // Changer le mot de passe
  db.getSiblingDB("admin").changeUserPassword(username, newPassword);

  // Logger le changement
  db.getSiblingDB("audit").password_rotations.insertOne({
    username: username,
    rotatedAt: new Date(),
    rotatedBy: db.runCommand({ connectionStatus: 1 }).authInfo.authenticatedUsers[0].user
  });

  // Notifier (via système externe)
  print(`Password rotated for ${username}`);
  print(`New password: ${newPassword}`);
  print(`Send to: ${notifyEmail}`);

  return newPassword;
}

function generateSecurePassword(length) {
  const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_+-=";
  let password = "";
  for (let i = 0; i < length; i++) {
    password += charset.charAt(Math.floor(Math.random() * charset.length));
  }
  return password;
}

// Usage
rotateUserPassword("webapp_prod", "devops@company.com");
```

### Modification des Rôles

#### Ajouter des Rôles

```javascript
use admin
db.grantRolesToUser("username", [
  { role: "dbAdmin", db: "production" },
  { role: "read", db: "analytics" }
])
```

#### Retirer des Rôles

```javascript
use admin
db.revokeRolesFromUser("username", [
  { role: "readWrite", db: "production" }
])
```

#### Remplacer Tous les Rôles

```javascript
use admin
db.updateUser("username", {
  roles: [
    { role: "read", db: "production" }  // Nouveaux rôles uniquement
  ]
})
```

### Modification des Métadonnées

```javascript
use admin
db.updateUser("webapp_prod", {
  customData: {
    lastReview: new Date(),
    reviewedBy: "security-team",
    status: "active",
    expirationDate: new Date("2026-12-31"),
    notes: "Annual security review completed"
  }
})
```

### Ajout de Restrictions d'Authentification

```javascript
use admin
db.updateUser("remote_analyst", {
  authenticationRestrictions: [
    {
      // Uniquement depuis VPN
      clientSource: ["203.0.113.0/24"],
      // Uniquement vers secondaires (lecture)
      serverAddress: ["10.0.1.101", "10.0.1.102"]
    }
  ]
})
```

### Activation/Désactivation (Soft Delete)

MongoDB ne supporte pas nativement la désactivation. Simulation :

```javascript
// Désactiver un utilisateur (retirer tous les rôles)
use admin
db.updateUser("username", {
  roles: [],  // Aucun rôle = pas d'accès
  customData: {
    status: "disabled",
    disabledAt: new Date(),
    disabledBy: "admin",
    reason: "Departed employee"
  }
})

// Réactiver
db.updateUser("username", {
  roles: [
    { role: "readWrite", db: "production" }
  ],
  customData: {
    status: "active",
    reactivatedAt: new Date(),
    reactivatedBy: "admin"
  }
})
```

## Suppression d'Utilisateurs

### Suppression Simple

```javascript
use admin
db.dropUser("username")
```

### Suppression Sécurisée avec Audit

```javascript
function secureDropUser(username, reason, ticket) {
  const user = db.getSiblingDB("admin").getUser(username);

  if (!user) {
    print(`❌ User not found: ${username}`);
    return;
  }

  // Audit avant suppression
  db.getSiblingDB("audit").user_deletions.insertOne({
    username: username,
    deletedAt: new Date(),
    deletedBy: db.runCommand({ connectionStatus: 1 }).authInfo.authenticatedUsers[0].user,
    reason: reason,
    ticket: ticket,
    userSnapshot: user  // Backup complet
  });

  // Suppression
  db.getSiblingDB("admin").dropUser(username);

  print(`✅ User deleted: ${username}`);
  print(`   Reason: ${reason}`);
  print(`   Ticket: ${ticket}`);
}

// Usage
secureDropUser("departed_employee", "Employee left company", "HR-2024-123");
```

### Suppression en Masse

```javascript
// Supprimer tous les utilisateurs désactivés
function cleanupDisabledUsers() {
  const users = db.getSiblingDB("admin").system.users.find({
    "customData.status": "disabled"
  });

  let count = 0;
  users.forEach(user => {
    const disabledDate = new Date(user.customData.disabledAt);
    const now = new Date();
    const daysSinceDisabled = (now - disabledDate) / (1000 * 60 * 60 * 24);

    // Supprimer après 90 jours
    if (daysSinceDisabled > 90) {
      secureDropUser(user.user, "Automatic cleanup - 90 days disabled", "AUTO");
      count++;
    }
  });

  print(`✅ Cleaned up ${count} disabled users`);
}
```

## Inspection et Audit des Utilisateurs

### Lister Tous les Utilisateurs

```javascript
use admin
db.getUsers()

// Avec les rôles et privilèges
db.getUsers({ showCredentials: false, showPrivileges: true })
```

### Voir un Utilisateur Spécifique

```javascript
use admin
db.getUser("username")

// Avec détails complets
db.getUser("username", {
  showCredentials: false,
  showPrivileges: true,
  showAuthenticationRestrictions: true
})
```

### Recherche d'Utilisateurs

```javascript
// Rechercher par rôle
function findUsersByRole(roleName, database) {
  const users = [];

  db.getSiblingDB("admin").system.users.find().forEach(user => {
    const hasRole = user.roles.some(r =>
      r.role === roleName && r.db === database
    );

    if (hasRole) {
      users.push({
        user: user.user,
        db: user.db,
        roles: user.roles
      });
    }
  });

  return users;
}

// Usage
print("Users with 'readWrite' on 'production':");
printjson(findUsersByRole("readWrite", "production"));

// Rechercher par métadonnées
function findUsersByCustomData(field, value) {
  const query = {};
  query[`customData.${field}`] = value;

  return db.getSiblingDB("admin").system.users.find(query).toArray();
}

// Usage
printjson(findUsersByCustomData("team", "Engineering"));
printjson(findUsersByCustomData("environment", "production"));
```

### Rapport d'Audit Complet

```javascript
function generateUserAuditReport() {
  const report = {
    generatedAt: new Date(),
    totalUsers: 0,
    byType: {},
    byEnvironment: {},
    expiringSoon: [],
    noRecentActivity: [],
    highPrivilegeUsers: []
  };

  const users = db.getSiblingDB("admin").system.users.find().toArray();
  report.totalUsers = users.length;

  users.forEach(user => {
    // Par type
    const type = user.customData?.accountType || "unknown";
    report.byType[type] = (report.byType[type] || 0) + 1;

    // Par environnement
    const env = user.customData?.environment || "unknown";
    report.byEnvironment[env] = (report.byEnvironment[env] || 0) + 1;

    // Expiration proche (30 jours)
    if (user.customData?.expirationDate) {
      const expirationDate = new Date(user.customData.expirationDate);
      const daysUntilExpiry = (expirationDate - new Date()) / (1000 * 60 * 60 * 24);

      if (daysUntilExpiry > 0 && daysUntilExpiry <= 30) {
        report.expiringSoon.push({
          user: user.user,
          expiresIn: Math.floor(daysUntilExpiry) + " days"
        });
      }
    }

    // Privilèges élevés
    const highPrivRoles = ["root", "dbOwner", "userAdminAnyDatabase", "dbAdminAnyDatabase"];
    const hasHighPriv = user.roles.some(r => highPrivRoles.includes(r.role));

    if (hasHighPriv) {
      report.highPrivilegeUsers.push({
        user: user.user,
        roles: user.roles
      });
    }
  });

  return report;
}

// Générer et afficher le rapport
const report = generateUserAuditReport();
printjson(report);
```

## Gestion du Cycle de Vie

### Onboarding d'Utilisateur

```javascript
class UserOnboarding {
  constructor(database) {
    this.db = db.getSiblingDB(database);
  }

  createUser(userData) {
    // Validation
    if (!userData.username || !userData.email || !userData.team) {
      throw new Error("Missing required fields");
    }

    // Générer mot de passe temporaire
    const tempPassword = this.generateTempPassword();

    // Créer l'utilisateur
    this.db.createUser({
      user: userData.username,
      pwd: tempPassword,
      roles: userData.roles || [],
      customData: {
        fullName: userData.fullName,
        email: userData.email,
        team: userData.team,
        manager: userData.manager,
        startDate: new Date(),
        status: "active",
        mustChangePassword: true,
        createdBy: this.getCurrentUser()
      }
    });

    // Logger l'onboarding
    this.db.getSiblingDB("audit").user_onboarding.insertOne({
      username: userData.username,
      onboardedAt: new Date(),
      onboardedBy: this.getCurrentUser(),
      initialRoles: userData.roles
    });

    // Retourner les credentials (à envoyer de manière sécurisée)
    return {
      username: userData.username,
      tempPassword: tempPassword,
      message: "User created. Temporary password must be changed on first login."
    };
  }

  generateTempPassword() {
    const length = 20;
    const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*";
    let password = "";
    for (let i = 0; i < length; i++) {
      password += charset.charAt(Math.floor(Math.random() * charset.length));
    }
    return password;
  }

  getCurrentUser() {
    const status = this.db.runCommand({ connectionStatus: 1 });
    return status.authInfo.authenticatedUsers[0]?.user || "unknown";
  }
}

// Usage
const onboarding = new UserOnboarding("admin");

const newUser = onboarding.createUser({
  username: "jane.smith",
  fullName: "Jane Smith",
  email: "jane.smith@company.com",
  team: "Engineering",
  manager: "john.manager@company.com",
  roles: [
    { role: "readWrite", db: "development" }
  ]
});

print(`User created: ${newUser.username}`);
print(`Temporary password: ${newUser.tempPassword}`);
print(`Send credentials securely to: jane.smith@company.com`);
```

### Offboarding d'Utilisateur

```javascript
class UserOffboarding {
  constructor(database) {
    this.db = db.getSiblingDB(database);
  }

  offboardUser(username, reason, ticket) {
    // Récupérer info utilisateur
    const user = this.db.getUser(username);

    if (!user) {
      throw new Error(`User not found: ${username}`);
    }

    // Phase 1 : Désactiver (retirer rôles)
    print(`Phase 1: Disabling user ${username}...`);
    this.db.updateUser(username, {
      roles: [],
      customData: {
        ...user.customData,
        status: "offboarded",
        offboardedAt: new Date(),
        offboardedBy: this.getCurrentUser(),
        reason: reason,
        ticket: ticket
      }
    });

    // Logger l'offboarding
    this.db.getSiblingDB("audit").user_offboarding.insertOne({
      username: username,
      offboardedAt: new Date(),
      offboardedBy: this.getCurrentUser(),
      reason: reason,
      ticket: ticket,
      previousRoles: user.roles,
      userSnapshot: user
    });

    print(`✅ User ${username} offboarded`);
    print(`   Status: Disabled (scheduled for deletion in 90 days)`);
    print(`   Reason: ${reason}`);
    print(`   Ticket: ${ticket}`);

    return {
      username: username,
      status: "offboarded",
      deletionScheduled: new Date(Date.now() + 90 * 24 * 60 * 60 * 1000)
    };
  }

  getCurrentUser() {
    const status = this.db.runCommand({ connectionStatus: 1 });
    return status.authInfo.authenticatedUsers[0]?.user || "unknown";
  }
}

// Usage
const offboarding = new UserOffboarding("admin");
offboarding.offboardUser(
  "departed.employee",
  "Employee left company",
  "HR-2024-456"
);
```

### Révision Périodique

```javascript
function quarterlyUserReview() {
  print("\n=== Quarterly User Access Review ===\n");

  const users = db.getSiblingDB("admin").system.users.find().toArray();
  const report = {
    date: new Date(),
    totalUsers: users.length,
    needsReview: [],
    needsRotation: [],
    expiringSoon: [],
    inactive: []
  };

  users.forEach(user => {
    const customData = user.customData || {};

    // Vérifier date de révision
    if (customData.reviewDate) {
      const reviewDate = new Date(customData.reviewDate);
      if (reviewDate < new Date()) {
        report.needsReview.push({
          user: user.user,
          lastReview: reviewDate
        });
      }
    } else {
      report.needsReview.push({
        user: user.user,
        lastReview: "never"
      });
    }

    // Vérifier rotation password
    if (customData.accountType === "service" && customData.nextRotation) {
      const nextRotation = new Date(customData.nextRotation);
      if (nextRotation < new Date()) {
        report.needsRotation.push({
          user: user.user,
          nextRotation: nextRotation
        });
      }
    }

    // Vérifier expiration
    if (customData.expirationDate) {
      const expDate = new Date(customData.expirationDate);
      const daysLeft = (expDate - new Date()) / (1000 * 60 * 60 * 24);
      if (daysLeft > 0 && daysLeft <= 30) {
        report.expiringSoon.push({
          user: user.user,
          daysLeft: Math.floor(daysLeft)
        });
      }
    }
  });

  // Afficher le rapport
  print(`Total users: ${report.totalUsers}`);
  print(`\nNeeds review (${report.needsReview.length}):`);
  report.needsReview.forEach(u => print(`  - ${u.user} (last: ${u.lastReview})`));

  print(`\nNeeds password rotation (${report.needsRotation.length}):`);
  report.needsRotation.forEach(u => print(`  - ${u.user}`));

  print(`\nExpiring soon (${report.expiringSoon.length}):`);
  report.expiringSoon.forEach(u => print(`  - ${u.user} (${u.daysLeft} days)`));

  return report;
}

// Exécuter révision
const reviewReport = quarterlyUserReview();
```

## Bonnes Pratiques

### 1. Convention de Nommage

```javascript
// ✅ BON : Noms explicites et cohérents
"john.doe"              // Humains : prénom.nom
"webapp_prod_001"       // Services : service_env_version
"svc_backup"            // Services : svc_fonction
"api_readonly_v2"       // APIs : api_permission_version

// ❌ MAUVAIS : Noms vagues
"user1"
"temp"
"test"
"admin123"
```

**Conventions recommandées** :

| Type | Format | Exemple |
|------|--------|---------|
| Humain | `prenom.nom` | `john.doe` |
| Service | `svc_fonction` | `svc_backup`, `svc_monitoring` |
| Application | `app_nom_env` | `webapp_prod`, `api_staging` |
| Batch | `batch_fonction` | `batch_etl`, `batch_report` |
| API | `api_version` | `api_v1`, `api_v2` |

### 2. Documentation Obligatoire

```javascript
// Toujours inclure customData avec métadonnées complètes
db.createUser({
  user: "new_service",
  pwd: passwordPrompt(),
  roles: [...],

  customData: {
    // Obligatoire
    fullName: "Service Description",
    owner: "team@company.com",
    createdDate: new Date(),
    purpose: "Detailed purpose",

    // Recommandé
    team: "Team Name",
    environment: "production|staging|development",
    accountType: "human|service|api",

    // Optionnel mais utile
    manager: "manager@company.com",
    project: "Project Name",
    costCenter: "CC-1234",
    ticketNumber: "JIRA-5678",
    approvedBy: "security-team"
  }
})
```

### 3. Rotation des Mots de Passe

```javascript
// Politique de rotation
const ROTATION_POLICIES = {
  human: {
    interval: 90,  // jours
    notifyBefore: 14  // jours
  },
  service: {
    interval: 180,  // jours
    notifyBefore: 30  // jours
  },
  privileged: {
    interval: 30,  // jours
    notifyBefore: 7  // jours
  }
};

function enforcePasswordRotation() {
  const users = db.getSiblingDB("admin").system.users.find().toArray();

  users.forEach(user => {
    const customData = user.customData || {};
    const accountType = customData.accountType || "human";
    const lastRotation = customData.lastPasswordChange || customData.createdDate;

    if (!lastRotation) return;

    const policy = ROTATION_POLICIES[accountType] || ROTATION_POLICIES.human;
    const daysSinceRotation = (new Date() - new Date(lastRotation)) / (1000 * 60 * 60 * 24);

    if (daysSinceRotation > policy.interval) {
      print(`⚠️  Password rotation overdue: ${user.user}`);
      print(`   Last rotation: ${Math.floor(daysSinceRotation)} days ago`);

      // Désactiver le compte
      db.getSiblingDB("admin").updateUser(user.user, {
        roles: [],
        customData: {
          ...customData,
          status: "locked",
          reason: "Password rotation overdue"
        }
      });
    } else if (daysSinceRotation > (policy.interval - policy.notifyBefore)) {
      const daysLeft = policy.interval - daysSinceRotation;
      print(`📧 Notify ${user.user}: Password expires in ${Math.floor(daysLeft)} days`);
    }
  });
}
```

### 4. Séparation des Privilèges

```javascript
// ❌ MAUVAIS : Un compte pour tout
db.createUser({
  user: "app_all",
  pwd: passwordPrompt(),
  roles: [
    { role: "root", db: "admin" }  // Trop de privilèges
  ]
})

// ✅ BON : Comptes séparés par fonction
db.createUser({
  user: "app_read",
  pwd: passwordPrompt(),
  roles: [{ role: "read", db: "production" }]
})

db.createUser({
  user: "app_write",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "production" }]
})

db.createUser({
  user: "app_admin",
  pwd: passwordPrompt(),
  roles: [{ role: "dbAdmin", db: "production" }]
})
```

### 5. Restrictions d'Accès

```javascript
// Toujours limiter par IP si possible
db.createUser({
  user: "remote_analyst",
  pwd: passwordPrompt(),
  roles: [{ role: "read", db: "production" }],

  authenticationRestrictions: [
    {
      clientSource: [
        "203.0.113.0/24",  // Bureau principal
        "198.51.100.0/24"  // Bureau secondaire
      ]
    }
  ]
})
```

### 6. Comptes de Service avec Rotation Automatique

```javascript
// Créer avec info de rotation
db.createUser({
  user: "svc_api_v1",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "production" }],

  customData: {
    accountType: "service",
    rotationPolicy: "automated",
    rotationInterval: 180,  // jours
    rotationMethod: "vault",  // HashiCorp Vault
    vaultPath: "secret/mongodb/svc_api_v1",
    lastRotation: new Date(),
    nextRotation: new Date(Date.now() + 180 * 24 * 60 * 60 * 1000)
  }
})
```

### 7. Audit et Traçabilité

```javascript
// Wrapper pour toutes les opérations utilisateur
function auditedCreateUser(userData) {
  const result = db.createUser(userData);

  // Logger dans collection d'audit
  db.getSiblingDB("audit").user_operations.insertOne({
    operation: "create",
    timestamp: new Date(),
    actor: db.runCommand({ connectionStatus: 1 }).authInfo.authenticatedUsers[0].user,
    target: userData.user,
    roles: userData.roles,
    customData: userData.customData
  });

  return result;
}

function auditedDropUser(username) {
  const user = db.getUser(username);

  const result = db.dropUser(username);

  db.getSiblingDB("audit").user_operations.insertOne({
    operation: "delete",
    timestamp: new Date(),
    actor: db.runCommand({ connectionStatus: 1 }).authInfo.authenticatedUsers[0].user,
    target: username,
    userSnapshot: user
  });

  return result;
}
```

## Automatisation

### Script Bash de Gestion

```bash
#!/bin/bash
# manage-mongodb-user.sh

MONGO_HOST="localhost:27017"
MONGO_AUTH_DB="admin"
MONGO_ADMIN_USER="admin"

function create_user() {
  local username=$1
  local roles=$2
  local db=$3

  mongosh "mongodb://${MONGO_ADMIN_USER}@${MONGO_HOST}/?authSource=${MONGO_AUTH_DB}" <<EOF
use admin
db.createUser({
  user: "${username}",
  pwd: passwordPrompt(),
  roles: ${roles},
  customData: {
    createdDate: new Date(),
    createdBy: "${MONGO_ADMIN_USER}",
    scriptVersion: "1.0"
  }
})
EOF

  echo "✅ User ${username} created"
}

function delete_user() {
  local username=$1

  echo "⚠️  Deleting user: ${username}"
  read -p "Confirm deletion (yes/no): " confirm

  if [ "$confirm" = "yes" ]; then
    mongosh "mongodb://${MONGO_ADMIN_USER}@${MONGO_HOST}/?authSource=${MONGO_AUTH_DB}" <<EOF
use admin
db.dropUser("${username}")
EOF
    echo "✅ User ${username} deleted"
  else
    echo "❌ Deletion cancelled"
  fi
}

function rotate_password() {
  local username=$1

  mongosh "mongodb://${MONGO_ADMIN_USER}@${MONGO_HOST}/?authSource=${MONGO_AUTH_DB}" <<EOF
use admin
db.changeUserPassword("${username}", passwordPrompt())
db.updateUser("${username}", {
  customData: {
    lastPasswordChange: new Date()
  }
})
EOF

  echo "✅ Password rotated for ${username}"
}

function list_users() {
  mongosh "mongodb://${MONGO_ADMIN_USER}@${MONGO_HOST}/?authSource=${MONGO_AUTH_DB}" <<EOF
use admin
db.getUsers().forEach(user => {
  print(\`User: \${user.user}\`);
  print(\`  Roles: \${JSON.stringify(user.roles)}\`);
  if (user.customData) {
    print(\`  Created: \${user.customData.createdDate}\`);
  }
  print("");
})
EOF
}

# Menu
case "$1" in
  create)
    create_user "$2" "$3" "$4"
    ;;
  delete)
    delete_user "$2"
    ;;
  rotate)
    rotate_password "$2"
    ;;
  list)
    list_users
    ;;
  *)
    echo "Usage: $0 {create|delete|rotate|list} [args]"
    exit 1
    ;;
esac
```

### Ansible Playbook

```yaml
# playbooks/mongodb-user-management.yml
---
- name: Manage MongoDB Users
  hosts: mongodb_primary
  vars:
    mongodb_admin_user: admin
    mongodb_admin_password: "{{ vault_mongodb_admin_password }}"
    mongodb_users:
      - username: webapp_prod
        password: "{{ vault_webapp_prod_password }}"
        roles:
          - role: readWrite
            db: production
        custom_data:
          team: Engineering
          environment: production

      - username: backup_service
        password: "{{ vault_backup_password }}"
        roles:
          - role: backup
            db: admin
          - role: clusterMonitor
            db: admin
        custom_data:
          accountType: service
          team: Operations

  tasks:
    - name: Create MongoDB users
      mongodb_user:
        login_user: "{{ mongodb_admin_user }}"
        login_password: "{{ mongodb_admin_password }}"
        database: admin
        name: "{{ item.username }}"
        password: "{{ item.password }}"
        roles: "{{ item.roles }}"
        state: present
      loop: "{{ mongodb_users }}"
      no_log: true  # Ne pas logger les passwords

    - name: Update user custom data
      shell: |
        mongosh --quiet --eval '
          db.getSiblingDB("admin").updateUser("{{ item.username }}", {
            customData: {{ item.custom_data | to_json }}
          })
        '
      loop: "{{ mongodb_users }}"
```

### Terraform pour MongoDB Atlas

```hcl
# mongodb-atlas-users.tf

variable "mongodb_users" {
  type = map(object({
    roles = list(object({
      role_name     = string
      database_name = string
    }))
    labels = map(string)
  }))
}

resource "mongodbatlas_database_user" "users" {
  for_each = var.mongodb_users

  username           = each.key
  password           = random_password.user_passwords[each.key].result
  project_id         = var.project_id
  auth_database_name = "admin"

  dynamic "roles" {
    for_each = each.value.roles
    content {
      role_name     = roles.value.role_name
      database_name = roles.value.database_name
    }
  }

  dynamic "labels" {
    for_each = each.value.labels
    content {
      key   = labels.key
      value = labels.value
    }
  }
}

resource "random_password" "user_passwords" {
  for_each = var.mongodb_users

  length  = 32
  special = true
}

# Stocker dans Vault
resource "vault_generic_secret" "mongodb_passwords" {
  for_each = var.mongodb_users

  path = "secret/mongodb/${each.key}"

  data_json = jsonencode({
    username = each.key
    password = random_password.user_passwords[each.key].result
  })
}

# terraform.tfvars
mongodb_users = {
  "webapp_prod" = {
    roles = [
      {
        role_name     = "readWrite"
        database_name = "production"
      }
    ]
    labels = {
      environment = "production"
      team        = "engineering"
    }
  }

  "backup_service" = {
    roles = [
      {
        role_name     = "backup"
        database_name = "admin"
      }
    ]
    labels = {
      accountType = "service"
      team        = "operations"
    }
  }
}
```

## Sécurité des Credentials

### Intégration HashiCorp Vault

```python
#!/usr/bin/env python3
# mongodb_vault_integration.py

import hvac
from pymongo import MongoClient
import os

class MongoDBVaultManager:
    def __init__(self, vault_url, vault_token):
        self.vault_client = hvac.Client(url=vault_url, token=vault_token)

    def create_user_with_vault(self, username, roles, custom_data):
        """Créer un utilisateur et stocker le password dans Vault"""

        # Générer un password fort
        password = self.generate_secure_password()

        # Créer l'utilisateur dans MongoDB
        mongo_client = MongoClient(
            os.getenv('MONGODB_URI'),
            username=os.getenv('MONGODB_ADMIN_USER'),
            password=os.getenv('MONGODB_ADMIN_PASSWORD')
        )

        mongo_client.admin.command(
            'createUser',
            username,
            pwd=password,
            roles=roles,
            customData=custom_data
        )

        # Stocker dans Vault
        self.vault_client.secrets.kv.v2.create_or_update_secret(
            path=f'mongodb/{username}',
            secret={
                'username': username,
                'password': password,
                'created_at': str(datetime.now()),
                'roles': roles
            }
        )

        print(f"✅ User {username} created, credentials stored in Vault")
        return username

    def rotate_password(self, username):
        """Rotation automatique du password"""

        # Générer nouveau password
        new_password = self.generate_secure_password()

        # Récupérer credentials admin depuis Vault
        admin_creds = self.vault_client.secrets.kv.v2.read_secret_version(
            path='mongodb/admin'
        )['data']['data']

        # Changer le password dans MongoDB
        mongo_client = MongoClient(
            os.getenv('MONGODB_URI'),
            username=admin_creds['username'],
            password=admin_creds['password']
        )

        mongo_client.admin.command(
            'updateUser',
            username,
            pwd=new_password
        )

        # Mettre à jour dans Vault
        current_secret = self.vault_client.secrets.kv.v2.read_secret_version(
            path=f'mongodb/{username}'
        )['data']['data']

        self.vault_client.secrets.kv.v2.create_or_update_secret(
            path=f'mongodb/{username}',
            secret={
                **current_secret,
                'password': new_password,
                'rotated_at': str(datetime.now())
            }
        )

        print(f"✅ Password rotated for {username}")

    def generate_secure_password(self, length=32):
        """Générer un password sécurisé"""
        import secrets
        import string

        alphabet = string.ascii_letters + string.digits + string.punctuation
        password = ''.join(secrets.choice(alphabet) for _ in range(length))
        return password

# Usage
if __name__ == '__main__':
    vault_mgr = MongoDBVaultManager(
        vault_url='https://vault.company.com',
        vault_token=os.getenv('VAULT_TOKEN')
    )

    # Créer utilisateur
    vault_mgr.create_user_with_vault(
        username='new_service',
        roles=[{'role': 'readWrite', 'db': 'production'}],
        custom_data={'team': 'engineering'}
    )

    # Rotation
    vault_mgr.rotate_password('webapp_prod')
```

### AWS Secrets Manager

```python
#!/usr/bin/env python3
# mongodb_aws_secrets.py

import boto3
import json
from pymongo import MongoClient

class MongoDBSecretsManager:
    def __init__(self, region_name='us-east-1'):
        self.secrets_client = boto3.client('secretsmanager', region_name=region_name)

    def create_user_with_secrets(self, username, roles, custom_data):
        """Créer utilisateur et stocker dans AWS Secrets Manager"""

        import secrets
        import string

        # Générer password
        password = ''.join(secrets.choice(
            string.ascii_letters + string.digits + string.punctuation
        ) for _ in range(32))

        # Créer dans MongoDB
        mongo_client = self.get_admin_connection()
        mongo_client.admin.command(
            'createUser',
            username,
            pwd=password,
            roles=roles,
            customData=custom_data
        )

        # Stocker dans Secrets Manager
        secret_value = {
            'username': username,
            'password': password,
            'roles': roles,
            'mongodb_uri': f"mongodb://{username}:{password}@mongodb.company.com:27017"
        }

        self.secrets_client.create_secret(
            Name=f'mongodb/{username}',
            SecretString=json.dumps(secret_value),
            Tags=[
                {'Key': 'Application', 'Value': 'MongoDB'},
                {'Key': 'Environment', 'Value': custom_data.get('environment', 'unknown')}
            ]
        )

        print(f"✅ User {username} created, secret stored in AWS Secrets Manager")

    def get_admin_connection(self):
        """Récupérer connexion admin depuis Secrets Manager"""
        secret = self.secrets_client.get_secret_value(SecretId='mongodb/admin')
        creds = json.loads(secret['SecretString'])

        return MongoClient(
            f"mongodb://{creds['username']}:{creds['password']}@mongodb.company.com:27017"
        )
```

## Troubleshooting

### Problèmes Courants

#### 1. "User not found"

```javascript
// Vérifier dans quelle base l'utilisateur existe
function findUser(username) {
  const databases = db.adminCommand({ listDatabases: 1 }).databases;

  for (let dbInfo of databases) {
    const users = db.getSiblingDB(dbInfo.name).getUsers();
    const user = users.find(u => u.user === username);

    if (user) {
      print(`✅ Found user '${username}' in database '${dbInfo.name}'`);
      printjson(user);
      return;
    }
  }

  print(`❌ User '${username}' not found in any database`);
}

// Usage
findUser("myusername");
```

#### 2. "Authentication failed"

```javascript
// Diagnostic complet
function diagnoseAuthFailure(username, database) {
  print(`\n=== Authentication Diagnostic for ${username}@${database} ===\n`);

  // Vérifier si utilisateur existe
  const user = db.getSiblingDB(database).getUser(username);
  if (!user) {
    print(`❌ User ${username} does not exist in ${database}`);
    return;
  }

  print(`✅ User exists in ${database}`);
  print(`Roles: ${JSON.stringify(user.roles)}`);

  // Vérifier status
  if (user.customData?.status === "disabled") {
    print(`❌ User is disabled`);
    print(`   Reason: ${user.customData.reason}`);
    return;
  }

  // Vérifier expiration
  if (user.customData?.expirationDate) {
    const expDate = new Date(user.customData.expirationDate);
    if (expDate < new Date()) {
      print(`❌ User has expired`);
      print(`   Expired on: ${expDate}`);
      return;
    }
  }

  // Vérifier restrictions IP
  if (user.authenticationRestrictions) {
    print(`\n⚠️  Authentication restrictions in place:`);
    printjson(user.authenticationRestrictions);
  }

  print(`\n✅ No obvious issues found`);
  print(`   Check: password correctness, network connectivity, authSource parameter`);
}

// Usage
diagnoseAuthFailure("webapp_prod", "admin");
```

#### 3. Permissions insuffisantes

```javascript
// Analyser les permissions d'un utilisateur
function analyzeUserPermissions(username, database) {
  const user = db.getSiblingDB(database).getUser(username, { showPrivileges: true });

  if (!user) {
    print(`User not found: ${username}@${database}`);
    return;
  }

  print(`\n=== Permissions Analysis for ${username} ===\n`);
  print(`Direct roles: ${user.roles.length}`);
  print(`Inherited roles: ${user.inheritedRoles.length}`);
  print(`Effective privileges: ${user.inheritedPrivileges.length}\n`);

  // Grouper par database
  const byDatabase = {};
  user.inheritedPrivileges.forEach(priv => {
    const db = priv.resource.db || "cluster";
    if (!byDatabase[db]) {
      byDatabase[db] = new Set();
    }
    priv.actions.forEach(action => byDatabase[db].add(action));
  });

  // Afficher
  Object.entries(byDatabase).forEach(([db, actions]) => {
    print(`Database: ${db}`);
    print(`  Actions: ${Array.from(actions).join(", ")}`);
  });
}

// Usage
analyzeUserPermissions("myuser", "admin");
```

## Conclusion

La gestion des utilisateurs MongoDB nécessite une approche structurée et automatisée pour garantir sécurité et opérabilité.

**Principes clés** :
- ✅ Conventions de nommage cohérentes et explicites
- ✅ Documentation complète via customData
- ✅ Séparation stricte des privilèges (moindre privilège)
- ✅ Rotation régulière des mots de passe
- ✅ Automatisation du cycle de vie (onboarding/offboarding)
- ✅ Audit systématique de toutes les opérations
- ✅ Stockage sécurisé des credentials (Vault, Secrets Manager)
- ✅ Révision périodique des accès
- ✅ Restrictions réseau quand possible
- ✅ Monitoring et alerting sur les changements

**Workflow recommandé** :
1. **Création** : Via scripts automatisés avec métadonnées complètes
2. **Provisioning** : Credentials stockés dans coffre-fort sécurisé
3. **Usage** : Audit continu, restrictions appliquées
4. **Révision** : Trimestrielle minimum
5. **Rotation** : Automatique selon politique
6. **Offboarding** : Désactivation immédiate, suppression différée

**Automatisation prioritaire** :
- Scripts de création/suppression standardisés
- Intégration avec Vault/Secrets Manager
- Playbooks Ansible/Terraform
- Monitoring des expirations
- Rotation automatique des comptes de service

Une gestion rigoureuse des utilisateurs est la pierre angulaire d'une infrastructure MongoDB sécurisée et conforme.

---

**Prochaines Sections du Chapitre 11 - Sécurité** :
- **11.5** : Chiffrement - TLS/SSL, chiffrement au repos, CSFLE
- **11.6** : Audit et conformité - Logs d'audit, traçabilité, RGPD
- **11.7** : Sécurité réseau - Firewalls, VPNs, IP whitelisting

⏭️ [Chiffrement](/11-securite/05-chiffrement.md)
