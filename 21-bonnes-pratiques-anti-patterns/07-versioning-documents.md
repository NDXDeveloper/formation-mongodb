🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.7 Versioning des Documents

## Introduction

Le versioning de documents est une exigence courante dans de nombreuses applications : audit, conformité réglementaire, collaboration, rollback, ou simplement historique des modifications. Cependant, implémenter le versioning dans MongoDB nécessite une réflexion approfondie sur la stratégie à adopter, car il n'existe pas de solution universelle. Un mauvais choix de stratégie peut mener à une explosion du stockage, des performances dégradées, ou une complexité applicative ingérable.

Cette section explore les différentes approches du versioning, leurs avantages et inconvénients, et fournit des patterns éprouvés pour implémenter un système de versioning robuste et performant adapté à vos besoins spécifiques.

---

## Comprendre les Besoins de Versioning

### Questions Clés à Se Poser

Avant d'implémenter un système de versioning, répondre à ces questions :

```javascript
const versioningRequirements = {
  // Quoi versionner ?
  scope: "document complet" | "champs spécifiques" | "actions métier",

  // Pourquoi versionner ?
  purpose: "audit" | "compliance" | "collaboration" | "undo/redo" | "historique",

  // Qui accède aux versions ?
  accessPattern: {
    current: "100%",      // Version actuelle
    previous: "10%",      // Version précédente
    history: "1%",        // Historique complet
    specific: "0.1%"      // Version spécifique
  },

  // Combien de temps conserver ?
  retention: "indefinite" | "1 year" | "90 days" | "30 versions",

  // Quelle granularité ?
  granularity: "field-level" | "document-level" | "operation-level",

  // Quel volume ?
  volumeEstimate: {
    documentsPerDay: 10000,
    changesPerDocument: 5,
    retentionDays: 365
    // = 18,250,000 versions par an
  }
};
```

---

## Stratégies de Versioning

### 1. Complete Snapshot (Copie Complète)

```javascript
// Version actuelle dans la collection principale
// Collection: documents
{
  _id: ObjectId("..."),
  title: "Contract v3",
  content: "Latest version content",
  version: 3,
  updatedAt: ISODate("2024-01-20T10:00:00Z")
}

// Versions historiques dans collection séparée
// Collection: documents_history
{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),  // Référence au document principal
  version: 1,
  title: "Contract v1",
  content: "First version content",
  createdAt: ISODate("2024-01-15T10:00:00Z")
}

{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),
  version: 2,
  title: "Contract v2",
  content: "Second version content",
  createdAt: ISODate("2024-01-18T10:00:00Z")
}
```

**Avantages** :
- Simple à implémenter et comprendre
- Chaque version est indépendante et complète
- Restauration rapide (copie directe)
- Requêtes simples pour accéder à n'importe quelle version

**Inconvénients** :
- Consommation d'espace importante (duplication)
- Coût d'écriture élevé (copie complète à chaque modification)

**Quand utiliser** :
- Documents petits (< 10 KB)
- Modifications peu fréquentes
- Besoin d'accès rapide à n'importe quelle version
- Conformité réglementaire stricte

### 2. Delta/Diff (Différences Incrémentales)

```javascript
// Version actuelle
// Collection: documents
{
  _id: ObjectId("..."),
  title: "Contract",
  content: "Current version with all updates",
  status: "approved",
  version: 3
}

// Deltas dans collection séparée
// Collection: documents_deltas
{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),
  version: 2,  // Cette delta permet de passer de v2 à v3
  timestamp: ISODate("2024-01-20T10:00:00Z"),
  changes: [
    {
      op: "replace",
      path: "/status",
      oldValue: "draft",
      newValue: "approved"
    },
    {
      op: "replace",
      path: "/content",
      oldValue: "Old content",
      newValue: "Current version with all updates"
    }
  ],
  author: "alice@company.com"
}
```

**Avantages** :
- Économie d'espace (seulement les changements)
- Audit précis (qui a changé quoi)
- Reconstruction possible de n'importe quelle version

**Inconvénients** :
- Reconstruction coûteuse (appliquer tous les deltas)
- Complexité d'implémentation
- Performance dégradée pour accès aux anciennes versions

**Quand utiliser** :
- Documents volumineux
- Modifications fréquentes mais mineures
- Besoin d'audit détaillé
- Accès rare aux anciennes versions

### 3. Embedded History (Historique Imbriqué)

```javascript
// Tout dans un seul document
{
  _id: ObjectId("..."),
  // Version actuelle
  current: {
    title: "Contract",
    content: "Latest version",
    status: "approved"
  },
  // Historique imbriqué
  history: [
    {
      version: 1,
      timestamp: ISODate("2024-01-15T10:00:00Z"),
      data: {
        title: "Contract",
        content: "First version",
        status: "draft"
      },
      author: "alice@company.com"
    },
    {
      version: 2,
      timestamp: ISODate("2024-01-18T10:00:00Z"),
      data: {
        title: "Contract",
        content: "Second version",
        status: "review"
      },
      author: "bob@company.com"
    }
  ],
  version: 3,
  updatedAt: ISODate("2024-01-20T10:00:00Z")
}
```

**Avantages** :
- Une seule lecture pour obtenir document + historique
- Pas de jointure nécessaire
- Atomicité garantie

**Inconvénients** :
- Document peut devenir très volumineux
- Risque de dépasser 16 MB
- Mise à jour du document entier à chaque modification

**Quand utiliser** :
- Historique limité (< 50 versions)
- Accès fréquent à l'historique récent
- Besoin d'atomicité stricte
- Documents avec peu de versions

---

## ✅ DO : Séparer Version Actuelle et Historique

**Explication** : Maintenir la version actuelle dans la collection principale et l'historique dans une collection dédiée optimise les performances.

**Pattern recommandé** :
```javascript
// ✅ Collection principale : seulement la version actuelle
// Collection: contracts
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  contractNumber: "CTR-2024-001",
  title: "Service Agreement",
  parties: ["Company A", "Company B"],
  amount: 50000,
  status: "active",
  // Métadonnées de version
  version: 5,
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-01-25T15:30:00Z"),
  createdBy: "alice@company.com",
  updatedBy: "bob@company.com"
}

// Collection séparée : historique complet
// Collection: contracts_versions
{
  _id: ObjectId("..."),
  contractId: ObjectId("507f1f77bcf86cd799439011"),
  version: 4,
  // Snapshot complet de la version précédente
  data: {
    contractNumber: "CTR-2024-001",
    title: "Service Agreement",
    parties: ["Company A", "Company B"],
    amount: 45000,  // Montant différent
    status: "draft"
  },
  // Métadonnées
  timestamp: ISODate("2024-01-20T10:00:00Z"),
  author: "alice@company.com",
  changeReason: "Amount adjustment",
  changeType: "update"
}
```

**Avantages mesurés** :

### 1. Performance des Requêtes Courantes
```javascript
// Requête de la version actuelle (99% des cas)
db.contracts.findOne({ _id: contractId });
// Temps : 1-2ms
// Document taille : 5 KB

// Si historique imbriqué
db.contracts.findOne({ _id: contractId });
// Temps : 10-20ms (document plus volumineux)
// Document taille : 50-100 KB avec 20 versions
= 5-10x plus lent
```

### 2. Économie d'Index
```javascript
// Index sur collection principale
db.contracts.createIndex({ status: 1, updatedAt: -1 });
// Taille index : Proportionnelle aux documents actuels

// Si historique imbriqué, index plus volumineux
// car documents plus gros
```

### 3. Scalabilité
```javascript
// Collection séparée permet :
// - TTL différent (expirer les anciennes versions)
// - Sharding différent
// - Backup sélectif (ne pas toujours backup l'historique)
// - Archivage vers storage froid
```

**Implémentation** :
```javascript
// ✅ Classe pour gérer le versioning
class VersionedDocument {
  constructor(collectionName) {
    this.collection = db[collectionName];
    this.versionsCollection = db[`${collectionName}_versions`];
  }

  async update(id, updates, metadata = {}) {
    // 1. Récupérer la version actuelle
    const current = await this.collection.findOne({ _id: id });

    if (!current) {
      throw new Error('Document not found');
    }

    // 2. Sauvegarder comme version historique
    await this.versionsCollection.insertOne({
      [`${collectionName.slice(0, -1)}Id`]: id,
      version: current.version,
      data: { ...current },
      timestamp: new Date(),
      author: metadata.author,
      changeReason: metadata.reason,
      changeType: 'update'
    });

    // 3. Mettre à jour le document principal
    const result = await this.collection.findOneAndUpdate(
      { _id: id },
      {
        $set: {
          ...updates,
          version: current.version + 1,
          updatedAt: new Date(),
          updatedBy: metadata.author
        }
      },
      { returnDocument: 'after' }
    );

    return result;
  }

  async getVersion(id, version) {
    if (version === 'latest') {
      return await this.collection.findOne({ _id: id });
    }

    return await this.versionsCollection.findOne({
      [`${this.collection.collectionName.slice(0, -1)}Id`]: id,
      version: version
    });
  }

  async getHistory(id, options = {}) {
    const { limit = 10, skip = 0 } = options;

    return await this.versionsCollection
      .find({ [`${this.collection.collectionName.slice(0, -1)}Id`]: id })
      .sort({ version: -1 })
      .skip(skip)
      .limit(limit)
      .toArray();
  }
}

// Usage
const contracts = new VersionedDocument('contracts');

await contracts.update(
  contractId,
  { amount: 55000, status: 'active' },
  { author: 'bob@company.com', reason: 'Final approval' }
);
```

---

## ❌ DON'T : Imbriquer un Historique Illimité

**Explication** : Stocker toutes les versions dans un tableau imbriqué mène rapidement à des documents dépassant la limite de 16 MB.

**Anti-pattern** :
```javascript
// ❌ Historique qui grossit indéfiniment
{
  _id: ObjectId("..."),
  title: "Long-lived Document",
  content: "Current version",
  history: [
    { version: 1, timestamp: "...", data: {...} },
    { version: 2, timestamp: "...", data: {...} },
    { version: 3, timestamp: "...", data: {...} },
    // ... après 100 modifications
    { version: 100, timestamp: "...", data: {...} }
  ]
}
// Taille du document : 2 MB et continue de croître
// Après 1000 modifications : 20 MB = ERREUR!
```

**Conséquences** :

### 1. Croissance Incontrôlée
```javascript
// Document initial : 5 KB
// Après 1 modification : 10 KB
// Après 10 modifications : 50 KB
// Après 100 modifications : 500 KB
// Après 500 modifications : 2.5 MB
// Après 1000 modifications : 5 MB (problématique)
// Après 3000 modifications : 15 MB (proche de la limite)
// Après 3200 modifications : 16 MB = CRASH!
```

### 2. Performance Dégradée
```javascript
// Temps de lecture
Document 5 KB : 1ms
Document 500 KB : 50ms
Document 5 MB : 500ms
= 500x plus lent

// Transfert réseau
Document 5 KB sur 100 Mbps : ~0.4ms
Document 5 MB sur 100 Mbps : ~400ms
= 1000x plus lent
```

### 3. Problèmes de Mise à Jour
```javascript
// Ajouter une nouvelle version = modifier document entier
// Si document = 5 MB
// Chaque ajout de version nécessite :
// - Lire 5 MB
// - Modifier en mémoire
// - Écrire 5 MB
// - Mettre à jour tous les index
```

---

## ✅ DO : Limiter l'Historique Imbriqué si Nécessaire

**Explication** : Si vous devez imbriquer l'historique, limitez-le aux N versions les plus récentes.

**Pattern recommandé** :
```javascript
// ✅ Historique limité avec $slice
{
  _id: ObjectId("..."),
  title: "Document",
  content: "Current version",
  version: 150,

  // Seulement les 20 dernières versions
  recentHistory: [
    { version: 131, timestamp: "...", data: {...} },
    { version: 132, timestamp: "...", data: {...} },
    // ...
    { version: 149, timestamp: "...", data: {...} }
  ],

  // Statistiques pour les anciennes versions
  totalVersions: 150,
  oldestVersionDate: ISODate("2023-01-15"),

  // Référence vers historique complet si nécessaire
  fullHistoryCollection: "documents_full_history"
}

// Mise à jour avec limitation
async function addVersion(docId, newData, metadata) {
  const current = await db.documents.findOne({ _id: docId });

  await db.documents.updateOne(
    { _id: docId },
    {
      $set: {
        ...newData,
        version: current.version + 1,
        updatedAt: new Date()
      },
      $push: {
        recentHistory: {
          $each: [{
            version: current.version,
            timestamp: new Date(),
            data: current,
            author: metadata.author
          }],
          $position: 0,    // Ajouter au début
          $slice: 20       // Garder seulement les 20 plus récents
        }
      },
      $inc: { totalVersions: 1 }
    }
  );

  // Sauvegarder dans l'historique complet
  await db.documents_full_history.insertOne({
    documentId: docId,
    version: current.version,
    data: current,
    timestamp: new Date(),
    author: metadata.author
  });
}
```

**Avantages** :
- Accès rapide aux versions récentes
- Taille de document contrôlée
- Historique complet préservé ailleurs
- Bon compromis performance/fonctionnalité

---

## ✅ DO : Utiliser des Stratégies de Compression pour l'Historique

**Explication** : Compresser les anciennes versions réduit significativement l'espace de stockage.

**Pattern recommandé** :
```javascript
// ✅ Versions récentes non compressées, anciennes compressées
const zlib = require('zlib');

async function saveVersion(docId, versionData, metadata) {
  const versionAge = Date.now() - versionData.timestamp;
  const oneDayMs = 24 * 60 * 60 * 1000;

  let dataToStore = versionData;

  // Compresser si version > 7 jours
  if (versionAge > 7 * oneDayMs) {
    const jsonString = JSON.stringify(versionData.data);
    const compressed = zlib.gzipSync(jsonString);

    dataToStore = {
      ...versionData,
      data: compressed,
      compressed: true,
      originalSize: jsonString.length,
      compressedSize: compressed.length,
      compressionRatio: (compressed.length / jsonString.length).toFixed(2)
    };
  }

  await db.documents_versions.insertOne(dataToStore);
}

// Récupération avec décompression
async function getVersion(docId, version) {
  const versionDoc = await db.documents_versions.findOne({
    documentId: docId,
    version: version
  });

  if (!versionDoc) return null;

  if (versionDoc.compressed) {
    const decompressed = zlib.gunzipSync(versionDoc.data);
    versionDoc.data = JSON.parse(decompressed.toString());
    delete versionDoc.compressed;
  }

  return versionDoc;
}
```

**Gains de compression mesurés** :
```javascript
// Documents JSON typiques
Original : 10 KB
Compressé : 2-3 KB
= 70-80% de réduction

// Documents avec beaucoup de texte
Original : 100 KB
Compressé : 10-15 KB
= 85-90% de réduction

// Pour 1M de versions de 10 KB chacune
Sans compression : 10 GB
Avec compression : 2-3 GB
= Économie de 7-8 GB
```

---

## ✅ DO : Implémenter un Système d'Audit avec les Versions

**Explication** : Le versioning est souvent lié à l'audit. Capturer les métadonnées complètes de chaque changement.

**Pattern complet d'audit** :
```javascript
// ✅ Métadonnées d'audit enrichies
{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),
  version: 42,

  // Données de la version
  data: { /* snapshot complet ou delta */ },

  // Métadonnées d'audit
  audit: {
    // Qui
    userId: ObjectId("..."),
    username: "alice@company.com",
    userRole: "editor",

    // Quand
    timestamp: ISODate("2024-01-20T15:30:00Z"),
    timezone: "Europe/Paris",

    // Quoi
    changeType: "update",  // create, update, delete, restore
    fieldsChanged: ["amount", "status"],
    changeReason: "Price adjustment per client request",
    changeCategory: "content",  // content, metadata, permission

    // Où
    ipAddress: "192.168.1.100",
    userAgent: "Mozilla/5.0...",
    application: "web-app",
    sessionId: "sess_abc123",

    // Pourquoi
    businessContext: {
      ticketId: "JIRA-1234",
      approvalId: "APR-5678",
      workflowStage: "review"
    },

    // Comment
    changes: [
      {
        field: "amount",
        oldValue: 45000,
        newValue: 50000,
        operation: "replace"
      },
      {
        field: "status",
        oldValue: "draft",
        newValue: "approved",
        operation: "replace"
      }
    ]
  },

  // Métadonnées techniques
  technical: {
    duration: 125,  // ms
    method: "API",
    endpoint: "/api/contracts/507f/update",
    serverInstance: "app-server-03"
  }
}
```

**Requêtes d'audit** :
```javascript
// ✅ Queries d'audit puissantes
// Qui a modifié ce document ?
db.documents_versions.distinct("audit.username", {
  documentId: docId
});

// Quels changements dans les dernières 24h ?
db.documents_versions.find({
  "audit.timestamp": {
    $gte: new Date(Date.now() - 24 * 60 * 60 * 1000)
  }
});

// Qui a changé ce champ spécifique ?
db.documents_versions.find({
  documentId: docId,
  "audit.fieldsChanged": "amount"
});

// Timeline des changements
db.documents_versions.aggregate([
  { $match: { documentId: docId } },
  { $sort: { version: 1 } },
  {
    $project: {
      version: 1,
      timestamp: "$audit.timestamp",
      author: "$audit.username",
      changes: "$audit.changes"
    }
  }
]);

// Rapport d'audit complet
db.documents_versions.aggregate([
  {
    $group: {
      _id: "$audit.username",
      modificationsCount: { $sum: 1 },
      lastModification: { $max: "$audit.timestamp" },
      documentsModified: { $addToSet: "$documentId" }
    }
  },
  { $sort: { modificationsCount: -1 } }
]);
```

---

## ❌ DON'T : Versionner Tout Sans Discernement

**Explication** : Versionner tous les documents et tous les champs crée une explosion de données et de complexité.

**Anti-pattern** :
```javascript
// ❌ Versioning aveugle de tout
// Collection: user_preferences_versions
{
  documentId: ObjectId("..."),
  version: 12547,  // 12,547 versions pour des préférences!
  data: {
    theme: "dark",
    language: "en",
    fontSize: 14
  },
  timestamp: ISODate("...")
}
// Stocker 12,000+ versions de préférences utilisateur
// qui changent constamment et dont l'historique n'a aucune valeur
```

**Problèmes** :
```javascript
// 1M d'utilisateurs
// Chaque utilisateur change ses préférences 100 fois
// = 100M de versions stockées
// Chaque version = 500 bytes
// Total : 50 GB de données inutiles

// Coût de stockage annuel pour des données sans valeur
```

**Solution - Versioning Sélectif** :
```javascript
// ✅ Versionner seulement ce qui a de la valeur
const versioningPolicy = {
  // Versionner : documents critiques
  contracts: {
    enabled: true,
    retention: "indefinite",
    reason: "Legal compliance"
  },

  financialRecords: {
    enabled: true,
    retention: "7 years",
    reason: "Regulatory requirement"
  },

  articles: {
    enabled: true,
    retention: "2 years",
    reason: "Editorial history"
  },

  // Ne pas versionner : données temporaires/non critiques
  userPreferences: {
    enabled: false,
    reason: "No business value in history"
  },

  sessionData: {
    enabled: false,
    reason: "Ephemeral data"
  },

  cache: {
    enabled: false,
    reason: "Temporary data"
  }
};

// Décision programmatique
function shouldVersion(collectionName, fieldName) {
  const policy = versioningPolicy[collectionName];

  if (!policy || !policy.enabled) {
    return false;
  }

  // Versionner seulement certains champs si spécifié
  if (policy.fieldsToVersion) {
    return policy.fieldsToVersion.includes(fieldName);
  }

  return true;
}
```

---

## ✅ DO : Implémenter un TTL pour les Anciennes Versions

**Explication** : Les anciennes versions peuvent être automatiquement supprimées après une période définie.

**Pattern recommandé** :
```javascript
// ✅ TTL sur l'historique avec politiques différenciées
// Index TTL basique
db.documents_versions.createIndex(
  { "timestamp": 1 },
  { expireAfterSeconds: 7776000 }  // 90 jours
);

// Politique plus sophistiquée avec flags
{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),
  version: 42,
  data: { /* ... */ },
  timestamp: ISODate("2024-01-20T15:30:00Z"),

  // Contrôle de rétention
  retention: {
    permanent: false,  // Si true, jamais supprimé
    expiresAt: ISODate("2024-04-20T15:30:00Z"),  // Date d'expiration
    category: "standard",  // standard, important, critical
    legalHold: false  // Si true, ne peut être supprimé
  }
}

// Index TTL conditionnel
db.documents_versions.createIndex(
  { "retention.expiresAt": 1 },
  {
    expireAfterSeconds: 0,
    partialFilterExpression: {
      "retention.permanent": { $ne: true },
      "retention.legalHold": { $ne: true }
    }
  }
);
```

**Politique de rétention par type** :
```javascript
// ✅ Différentes rétentions selon le contexte
const retentionPolicies = {
  contracts: {
    critical: "indefinite",  // Versions clés (signatures)
    standard: "7 years",     // Versions intermédiaires
    draft: "1 year"          // Brouillons
  },

  articles: {
    published: "2 years",
    draft: "90 days"
  },

  settings: {
    all: "30 days"
  }
};

async function saveVersionWithRetention(docId, versionData, metadata) {
  const retention = calculateRetention(
    metadata.collectionName,
    metadata.documentType,
    metadata.versionType
  );

  await db[`${metadata.collectionName}_versions`].insertOne({
    documentId: docId,
    version: versionData.version,
    data: versionData,
    timestamp: new Date(),
    retention: {
      permanent: retention === 'indefinite',
      expiresAt: retention !== 'indefinite'
        ? new Date(Date.now() + parseRetention(retention))
        : null,
      policy: retention,
      reason: metadata.reason
    }
  });
}
```

---

## ✅ DO : Permettre la Comparaison entre Versions

**Explication** : Fournir des outils pour comparer facilement deux versions d'un document.

**Implémentation de diff** :
```javascript
// ✅ Fonction de comparaison de versions
const jsondiffpatch = require('jsondiffpatch');

class VersionComparator {
  async compareVersions(docId, version1, version2) {
    // Récupérer les deux versions
    const v1 = await this.getVersion(docId, version1);
    const v2 = await this.getVersion(docId, version2);

    if (!v1 || !v2) {
      throw new Error('Version not found');
    }

    // Générer le diff
    const delta = jsondiffpatch.diff(v1.data, v2.data);

    return {
      version1: version1,
      version2: version2,
      delta: delta,
      summary: this.generateSummary(delta),
      html: jsondiffpatch.formatters.html.format(delta)
    };
  }

  generateSummary(delta) {
    const summary = {
      fieldsAdded: [],
      fieldsRemoved: [],
      fieldsModified: []
    };

    Object.keys(delta || {}).forEach(key => {
      const change = delta[key];

      if (Array.isArray(change)) {
        if (change.length === 1) {
          summary.fieldsAdded.push(key);
        } else if (change.length === 2) {
          summary.fieldsModified.push(key);
        } else if (change.length === 3 && change[2] === 0) {
          summary.fieldsRemoved.push(key);
        }
      }
    });

    return summary;
  }

  async getChangeHistory(docId) {
    const versions = await db.documents_versions
      .find({ documentId: docId })
      .sort({ version: 1 })
      .toArray();

    const history = [];

    for (let i = 1; i < versions.length; i++) {
      const previous = versions[i - 1];
      const current = versions[i];

      const diff = await this.compareVersions(
        docId,
        previous.version,
        current.version
      );

      history.push({
        from: previous.version,
        to: current.version,
        timestamp: current.timestamp,
        author: current.audit.username,
        changes: diff.summary
      });
    }

    return history;
  }
}

// Usage
const comparator = new VersionComparator();

// Comparer deux versions spécifiques
const diff = await comparator.compareVersions(docId, 5, 7);

console.log('Summary:', diff.summary);
// {
//   fieldsAdded: ['newField'],
//   fieldsRemoved: ['oldField'],
//   fieldsModified: ['amount', 'status']
// }

// Afficher le diff en HTML
res.send(diff.html);
```

---

## ✅ DO : Implémenter la Restauration de Versions

**Explication** : Permettre de restaurer facilement une version précédente d'un document.

**Pattern recommandé** :
```javascript
// ✅ Restauration sécurisée avec confirmation
class VersionRestorer {
  async restoreVersion(docId, targetVersion, metadata) {
    // 1. Vérifier que la version existe
    const versionToRestore = await db.documents_versions.findOne({
      documentId: docId,
      version: targetVersion
    });

    if (!versionToRestore) {
      throw new Error(`Version ${targetVersion} not found`);
    }

    // 2. Récupérer la version actuelle
    const current = await db.documents.findOne({ _id: docId });

    if (!current) {
      throw new Error('Document not found');
    }

    // 3. Créer une sauvegarde de la version actuelle
    await db.documents_versions.insertOne({
      documentId: docId,
      version: current.version,
      data: current,
      timestamp: new Date(),
      audit: {
        username: metadata.author,
        timestamp: new Date(),
        changeType: 'pre-restore-backup',
        changeReason: `Backup before restoring to version ${targetVersion}`
      }
    });

    // 4. Restaurer la version cible
    const restored = {
      ...versionToRestore.data,
      version: current.version + 1,  // Nouvelle version
      restoredFrom: targetVersion,
      updatedAt: new Date(),
      updatedBy: metadata.author
    };

    await db.documents.replaceOne(
      { _id: docId },
      restored
    );

    // 5. Enregistrer l'action de restauration
    await db.documents_versions.insertOne({
      documentId: docId,
      version: restored.version,
      data: restored,
      timestamp: new Date(),
      audit: {
        username: metadata.author,
        timestamp: new Date(),
        changeType: 'restore',
        changeReason: metadata.reason,
        restoredFromVersion: targetVersion
      }
    });

    return {
      success: true,
      restoredVersion: targetVersion,
      newVersion: restored.version,
      message: `Document restored to version ${targetVersion}`
    };
  }

  async previewRestore(docId, targetVersion) {
    // Montrer ce qui va changer sans l'appliquer
    const versionToRestore = await db.documents_versions.findOne({
      documentId: docId,
      version: targetVersion
    });

    const current = await db.documents.findOne({ _id: docId });

    const comparator = new VersionComparator();
    const diff = await comparator.compareVersions(
      docId,
      current.version,
      targetVersion
    );

    return {
      currentVersion: current.version,
      targetVersion: targetVersion,
      changes: diff.summary,
      warning: diff.summary.fieldsRemoved.length > 0
        ? 'This restore will remove some fields'
        : null
    };
  }
}

// Usage avec confirmation
async function restoreWithConfirmation(docId, targetVersion, userId) {
  const restorer = new VersionRestorer();

  // 1. Prévisualiser les changements
  const preview = await restorer.previewRestore(docId, targetVersion);

  console.log('Restore Preview:', preview);

  // 2. Demander confirmation utilisateur
  const confirmed = await askUserConfirmation(preview);

  if (!confirmed) {
    return { cancelled: true };
  }

  // 3. Effectuer la restauration
  return await restorer.restoreVersion(
    docId,
    targetVersion,
    {
      author: userId,
      reason: 'User-initiated restore'
    }
  );
}
```

---

## ❌ DON'T : Perdre les Métadonnées lors du Versioning

**Explication** : Versionner uniquement les données sans les métadonnées perd des informations contextuelles cruciales.

**Anti-pattern** :
```javascript
// ❌ Version sans contexte
{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),
  version: 5,
  data: {
    title: "Contract",
    amount: 50000
    // Juste les données, rien d'autre
  }
}

// Questions sans réponse :
// - Qui a fait ce changement ?
// - Pourquoi ?
// - Quand exactement ?
// - Depuis quelle application ?
// - Quels champs ont changé ?
```

**Solution complète** :
```javascript
// ✅ Version avec contexte complet
{
  _id: ObjectId("..."),
  documentId: ObjectId("..."),
  version: 5,

  // Données versionnées
  data: {
    title: "Contract",
    amount: 50000
  },

  // Contexte complet
  metadata: {
    // Qui
    author: {
      userId: ObjectId("..."),
      username: "alice@company.com",
      displayName: "Alice Smith",
      role: "editor"
    },

    // Quand
    timestamp: ISODate("2024-01-20T15:30:00Z"),

    // Quoi
    changeType: "update",
    fieldsChanged: ["amount"],
    previousValues: {
      amount: 45000
    },

    // Pourquoi
    changeReason: "Client negotiation result",
    businessContext: {
      ticketId: "JIRA-1234",
      approvalId: "APR-5678"
    },

    // Où/Comment
    source: {
      application: "web-app",
      ipAddress: "192.168.1.100",
      userAgent: "Mozilla/5.0..."
    }
  }
}
```

---

## Patterns Avancés

### ✅ DO : Implémenter Event Sourcing pour Cas Complexes

**Explication** : L'Event Sourcing stocke les événements (actions) plutôt que l'état, permettant une reconstruction complète de l'historique.

**Pattern Event Sourcing** :
```javascript
// ✅ Stream d'événements
// Collection: contract_events
{
  _id: ObjectId("..."),
  streamId: "contract-507f",  // Identifiant du stream
  eventType: "ContractCreated",
  eventNumber: 1,
  timestamp: ISODate("2024-01-15T10:00:00Z"),
  data: {
    contractNumber: "CTR-2024-001",
    parties: ["Company A", "Company B"],
    amount: 45000
  },
  metadata: {
    userId: "alice@company.com",
    correlationId: "req-123"
  }
}

{
  _id: ObjectId("..."),
  streamId: "contract-507f",
  eventType: "AmountAdjusted",
  eventNumber: 2,
  timestamp: ISODate("2024-01-18T14:00:00Z"),
  data: {
    oldAmount: 45000,
    newAmount: 50000,
    reason: "Client negotiation"
  },
  metadata: {
    userId: "bob@company.com",
    correlationId: "req-456"
  }
}

{
  _id: ObjectId("..."),
  streamId: "contract-507f",
  eventType: "ContractApproved",
  eventNumber: 3,
  timestamp: ISODate("2024-01-20T10:00:00Z"),
  data: {
    approvedBy: "manager@company.com",
    approvalDate: ISODate("2024-01-20T10:00:00Z")
  },
  metadata: {
    userId: "manager@company.com",
    correlationId: "req-789"
  }
}

// Index pour lecture efficace du stream
db.contract_events.createIndex({
  streamId: 1,
  eventNumber: 1
}, { unique: true });
```

**Reconstruction de l'état** :
```javascript
// ✅ Reconstruire l'état actuel depuis les événements
class EventStore {
  async getStream(streamId) {
    return await db.contract_events
      .find({ streamId: streamId })
      .sort({ eventNumber: 1 })
      .toArray();
  }

  async getCurrentState(streamId) {
    const events = await this.getStream(streamId);

    let state = {};

    for (const event of events) {
      state = this.applyEvent(state, event);
    }

    return state;
  }

  applyEvent(state, event) {
    switch (event.eventType) {
      case 'ContractCreated':
        return {
          ...event.data,
          status: 'draft',
          version: 1
        };

      case 'AmountAdjusted':
        return {
          ...state,
          amount: event.data.newAmount,
          version: state.version + 1
        };

      case 'ContractApproved':
        return {
          ...state,
          status: 'approved',
          approvedBy: event.data.approvedBy,
          approvedAt: event.data.approvalDate,
          version: state.version + 1
        };

      default:
        return state;
    }
  }

  async getStateAtVersion(streamId, targetVersion) {
    const events = await this.getStream(streamId);

    let state = {};
    let version = 0;

    for (const event of events) {
      if (version >= targetVersion) break;
      state = this.applyEvent(state, event);
      version++;
    }

    return state;
  }
}
```

**Snapshots pour performance** :
```javascript
// ✅ Snapshots périodiques pour éviter de rejouer tous les événements
// Collection: contract_snapshots
{
  _id: ObjectId("..."),
  streamId: "contract-507f",
  eventNumber: 100,  // État après 100 événements
  timestamp: ISODate("2024-01-25T10:00:00Z"),
  state: {
    // État complet à ce point
    contractNumber: "CTR-2024-001",
    amount: 50000,
    status: "approved",
    version: 100
  }
}

// Reconstruction optimisée
async function getCurrentStateOptimized(streamId) {
  // 1. Trouver le snapshot le plus récent
  const snapshot = await db.contract_snapshots
    .findOne({ streamId: streamId })
    .sort({ eventNumber: -1 });

  let state = snapshot ? snapshot.state : {};
  let fromEventNumber = snapshot ? snapshot.eventNumber + 1 : 1;

  // 2. Appliquer seulement les événements depuis le snapshot
  const recentEvents = await db.contract_events
    .find({
      streamId: streamId,
      eventNumber: { $gte: fromEventNumber }
    })
    .sort({ eventNumber: 1 })
    .toArray();

  for (const event of recentEvents) {
    state = applyEvent(state, event);
  }

  return state;
}

// Créer un snapshot tous les N événements
async function createSnapshotIfNeeded(streamId) {
  const eventCount = await db.contract_events.countDocuments({
    streamId: streamId
  });

  const lastSnapshot = await db.contract_snapshots
    .findOne({ streamId: streamId })
    .sort({ eventNumber: -1 });

  const lastSnapshotEventNumber = lastSnapshot
    ? lastSnapshot.eventNumber
    : 0;

  // Créer un snapshot tous les 100 événements
  if (eventCount - lastSnapshotEventNumber >= 100) {
    const currentState = await getCurrentState(streamId);

    await db.contract_snapshots.insertOne({
      streamId: streamId,
      eventNumber: eventCount,
      timestamp: new Date(),
      state: currentState
    });
  }
}
```

---

## Checklist Versioning

### Stratégie
- [ ] Besoins de versioning clairement définis
- [ ] Stratégie choisie (snapshot, delta, embedded)
- [ ] Pattern d'accès analysé (actuel vs historique)
- [ ] Volumétrie estimée
- [ ] Politique de rétention définie

### Implémentation
- [ ] Collection séparée pour versions (si applicable)
- [ ] Champ schemaVersion dans documents
- [ ] Métadonnées d'audit complètes
- [ ] Limitation de l'historique imbriqué (si applicable)
- [ ] Index appropriés créés

### Performance
- [ ] Compression pour anciennes versions (si applicable)
- [ ] TTL configuré pour rétention
- [ ] Snapshots pour Event Sourcing (si applicable)
- [ ] Monitoring de la taille des collections
- [ ] Archivage vers cold storage planifié

### Fonctionnalités
- [ ] Comparaison entre versions implémentée
- [ ] Restauration de version possible
- [ ] Prévisualisation de restauration
- [ ] Audit trail complet
- [ ] API de versioning documentée

### Maintenance
- [ ] Nettoyage automatique des anciennes versions
- [ ] Politique de rétention respectée
- [ ] Backup des versions critiques
- [ ] Documentation à jour

---

## Conclusion

Le versioning dans MongoDB nécessite :

- **Stratégie adaptée** : Snapshot, Delta ou Event Sourcing selon les besoins
- **Séparation** : Version actuelle vs historique
- **Audit complet** : Métadonnées riches pour traçabilité
- **Performance** : Compression, TTL, limitation
- **Fonctionnalités** : Comparaison, restauration, prévisualisation

**Règles d'or** :
1. **Séparer** : Version actuelle dans collection principale
2. **Limiter** : Historique imbriqué à 20-50 versions max
3. **Auditer** : Métadonnées complètes sur chaque version
4. **Sélectionner** : Versionner seulement ce qui a de la valeur
5. **Expirer** : TTL sur anciennes versions non critiques
6. **Comparer** : Outils de diff entre versions

Un bon système de versioning équilibre traçabilité, performance et coût de stockage. Trop de versioning = gaspillage, pas assez = perte d'information critique.

---


⏭️ [Gestion des environnements](/21-bonnes-pratiques-anti-patterns/08-gestion-environnements.md)
