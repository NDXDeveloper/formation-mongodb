🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.2 Gestion des _id

## Introduction

Le champ `_id` est l'élément le plus fondamental de MongoDB : il identifie de manière unique chaque document dans une collection. Bien que MongoDB offre une flexibilité totale sur le type et le format des `_id`, cette liberté peut devenir un piège. Un mauvais choix d'identifiant impacte la performance, la sécurité, la scalabilité et la maintenabilité du système.

Cette section explore les meilleures pratiques pour la gestion des `_id`, les différents types disponibles, leurs avantages et inconvénients, et surtout les pièges à éviter qui peuvent avoir des conséquences désastreuses en production.

---

## Comprendre le Champ _id

### Caractéristiques Fondamentales

Le champ `_id` dans MongoDB possède des propriétés uniques :

- **Obligatoire** : Chaque document doit avoir un `_id`
- **Unique** : Au sein d'une collection, chaque `_id` doit être unique
- **Immutable** : Une fois créé, l'`_id` ne peut pas être modifié
- **Indexé automatiquement** : MongoDB crée automatiquement un index unique sur `_id`
- **Type flexible** : Peut être de n'importe quel type BSON (sauf array)

### Impact de l'_id sur le Système

Le choix de l'`_id` influence :
- **Performance d'insertion** : Distribution des écritures dans le cluster
- **Performance de lecture** : Efficacité des requêtes par _id
- **Taille du stockage** : L'_id est dupliqué dans tous les index
- **Sécurité** : Exposition d'informations sensibles
- **Sharding** : Souvent utilisé comme shard key

---

## Types d'Identifiants

### ObjectId (Par Défaut)

```javascript
ObjectId("507f1f77bcf86cd799439011")
```

**Structure** :
- 4 bytes : Timestamp (secondes depuis epoch)
- 5 bytes : Valeur aléatoire (process unique)
- 3 bytes : Compteur incrémentiel

---

## ✅ DO : Utiliser ObjectId par Défaut

**Explication** : ObjectId est la solution recommandée pour la majorité des cas d'usage. Il est conçu spécifiquement pour MongoDB et offre un excellent équilibre entre unicité, performance et fonctionnalités.

**Avantages** :
```javascript
// ✅ MongoDB génère automatiquement
const result = await db.users.insertOne({
  name: "Alice",
  email: "alice@example.com"
  // _id généré automatiquement
});

console.log(result.insertedId);
// ObjectId("65a1b2c3d4e5f6789abcdef0")
```

**Bénéfices mesurables** :
- **Unicité garantie** : Même dans un système distribué
- **Ordonnancement chronologique** : Tri naturel par date de création
- **Performance optimale** : 12 bytes compacts
- **Extraction du timestamp** : `objectId.getTimestamp()`
- **Génération distribuée** : Pas de point de contention
- **Compatibilité** : Supporté par tous les drivers

**Cas d'usage idéaux** :
- Applications standard web/mobile
- Systèmes distribués
- Microservices
- Quand vous n'avez pas de contrainte spécifique

---

## ❌ DON'T : Rejeter ObjectId sans Raison Valable

**Explication** : Remplacer ObjectId par un système custom introduit de la complexité et des risques sans bénéfice réel dans 90% des cas.

**Anti-pattern courant** :
```javascript
// ❌ Réinventer la roue sans bénéfice
const userId = generateCustomId(); // UUID, ULID, ou autre
await db.users.insertOne({
  _id: userId,
  name: "Alice"
});
```

**Conséquences** :
- Perte de l'ordonnancement chronologique naturel
- Code de génération à maintenir
- Risque de collisions si mal implémenté
- Perte du timestamp embarqué
- Complexité accrue sans gain

**Raisons valables de ne pas utiliser ObjectId** :
1. Migration depuis un système existant avec IDs spécifiques
2. Besoins métier de format particulier (ex: numéro de commande)
3. Compatibilité avec système externe
4. Exigences de sécurité spécifiques (non-devinabilité)
5. Optimisation sharding avec shard key spécifique

Si vous n'avez aucune de ces raisons : **utilisez ObjectId**.

---

## ✅ DO : Utiliser UUID v4 pour la Non-Prédictibilité

**Explication** : Les UUIDs v4 (random) offrent une alternative valable quand la non-prédictibilité des IDs est une exigence de sécurité.

**Usage approprié** :
```javascript
// ✅ UUID pour des raisons de sécurité
import { v4 as uuidv4 } from 'uuid';

const sessionId = uuidv4();
await db.sessions.insertOne({
  _id: sessionId,  // "550e8400-e29b-41d4-a716-446655440000"
  userId: userId,
  createdAt: new Date()
});
```

**Avantages** :
- IDs non-séquentiels et non-devinables
- Standard universel (RFC 4122)
- Génération sans coordination
- Compatibilité cross-platform

**Quand utiliser** :
- Tokens de session
- URLs publiques (éviter l'énumération)
- Identifiants exposés dans les URLs
- Intégrations avec systèmes externes utilisant UUID

**Considérations** :
- **Taille** : 36 caractères en string (vs 24 pour ObjectId)
- **Performance** : Légèrement moins efficace que ObjectId
- **Stockage** : Utilisez le type BSON UUID/Binary pour économiser de l'espace

```javascript
// ✅ Stockage optimisé avec Binary
import { Binary } from 'mongodb';

const uuidBinary = new Binary(Buffer.from(uuidv4().replace(/-/g, ''), 'hex'), 4);
await db.sessions.insertOne({
  _id: uuidBinary,  // 16 bytes au lieu de 36 caractères
  userId: userId
});
```

---

## ❌ DON'T : Utiliser des Entiers Auto-Incrémentés

**Explication** : Les IDs auto-incrémentés (pattern SQL) sont un anti-pattern majeur dans MongoDB et les systèmes distribués.

**Anti-pattern** :
```javascript
// ❌ ANTI-PATTERN : Auto-increment dans MongoDB
async function getNextUserId() {
  const counter = await db.counters.findOneAndUpdate(
    { _id: "userId" },
    { $inc: { seq: 1 } },
    { returnDocument: 'after', upsert: true }
  );
  return counter.seq;
}

const newUserId = await getNextUserId();
await db.users.insertOne({
  _id: newUserId,  // 1, 2, 3, 4...
  name: "Alice"
});
```

**Conséquences désastreuses** :

### 1. Point de Contention Unique
- Toutes les insertions doivent passer par le compteur
- Bottleneck majeur de performance
- Impossible de scaler horizontalement

**Impact mesuré** :
```
ObjectId : 10,000 insertions/sec (distribué)
Auto-increment : 200-500 insertions/sec (bottleneck)
= 20-50x plus lent
```

### 2. Incompatible avec le Sharding
```javascript
// ❌ Shard key monotone = hot shard
sh.shardCollection("mydb.users", { _id: 1 })
// Toutes les insertions vont sur le même shard
// Les autres shards sont inutilisés
```

### 3. Problèmes de Sécurité
- IDs prédictibles et énumérables
- Exposition du volume de données (`userId: 150000` = 150k users)
- Vulnérabilité aux attaques par énumération

### 4. Problèmes de Synchronisation
- Conflits dans les environnements multi-master
- Complexité dans les architectures distribuées
- Impossible d'insérer offline et synchroniser

### 5. Perte de Bénéfices MongoDB
- Pas de timestamp embarqué
- Pas de génération distribuée
- Dépendance à un état global

**Alternatives légitimes** :
```javascript
// ✅ Si vous avez vraiment besoin d'un numéro séquentiel VISIBLE
{
  _id: ObjectId("..."),              // ID technique
  orderNumber: "ORD-2024-00001",     // Numéro métier séquentiel
  customerId: ObjectId("..."),
  total: 150.00
}
```

---

## ✅ DO : Utiliser des Identifiants Métier Naturels (Quand Approprié)

**Explication** : Si votre domaine possède déjà des identifiants naturels uniques et immuables, les utiliser comme `_id` peut simplifier le modèle.

**Cas d'usage valides** :
```javascript
// ✅ Email comme _id (si garantie d'unicité et immutabilité)
await db.users.insertOne({
  _id: "alice@example.com",  // Email = identifiant naturel
  name: "Alice Smith",
  registeredAt: new Date()
});

// ✅ Numéro de sécurité sociale (dans certains contextes)
await db.citizens.insertOne({
  _id: "123-45-6789",  // SSN = identifiant national unique
  name: "John Doe",
  birthDate: new Date("1990-01-15")
});

// ✅ Code produit normalisé (SKU, GTIN, ISBN)
await db.products.insertOne({
  _id: "ISBN-978-0-7475-3269-9",  // ISBN = standard international
  title: "Harry Potter and the Philosopher's Stone",
  author: "J.K. Rowling"
});

// ✅ Coordonnées géographiques uniques
await db.locations.insertOne({
  _id: "48.8566_2.3522",  // lat_lon
  city: "Paris",
  country: "France"
});
```

**Critères pour utiliser un identifiant naturel** :

✅ **DOIT** :
- Être unique par définition
- Être immuable (ne change jamais)
- Exister au moment de la création
- Être relativement compact

❌ **NE DOIT PAS** :
- Pouvoir changer (ex: username, phone)
- Dépendre de données externes
- Être trop long (>100 caractères)

---

## ❌ DON'T : Utiliser des Données Mutables comme _id

**Explication** : Utiliser des données qui peuvent changer comme `_id` crée des problèmes insurmontables car l'`_id` est immutable.

**Anti-patterns critiques** :
```javascript
// ❌ Username comme _id (peut changer)
await db.users.insertOne({
  _id: "alice2024",  // Et si Alice veut changer son username?
  email: "alice@example.com"
});
// Impossible de changer l'_id ensuite!

// ❌ Email comme _id (peut changer)
await db.accounts.insertOne({
  _id: "temp@example.com",  // Et si l'utilisateur change d'email?
  name: "Alice"
});

// ❌ Numéro de téléphone comme _id
await db.contacts.insertOne({
  _id: "+33612345678",  // Peut changer ou être réattribué
  name: "Alice"
});

// ❌ Données composites qui peuvent évoluer
await db.employees.insertOne({
  _id: "DEPT-IT-0042",  // Et si l'employé change de département?
  name: "Bob"
});
```

**Conséquences** :
- **Impossible de modifier** : L'_id est immutable par design
- **Migration forcée** : Seule solution = créer nouveau document et migrer toutes les références
- **Références cassées** : Tous les documents référençant cet _id deviennent invalides
- **Complexité exponentielle** : Impact en cascade sur tout le système

**Solution** :
```javascript
// ✅ Séparer _id technique et identifiant métier
await db.users.insertOne({
  _id: ObjectId("..."),        // ID technique immutable
  username: "alice2024",       // Peut changer
  email: "alice@example.com",  // Peut changer
  phone: "+33612345678"        // Peut changer
});
// Créer des index uniques sur les champs métier si nécessaire
db.users.createIndex({ username: 1 }, { unique: true });
db.users.createIndex({ email: 1 }, { unique: true });
```

---

## ✅ DO : Générer des _id Côté Application pour le Contrôle

**Explication** : Générer l'`_id` côté application avant l'insertion offre plus de contrôle et permet des opérations avancées.

**Avantages** :
```javascript
// ✅ Génération côté application
import { ObjectId } from 'mongodb';

const userId = new ObjectId();
const profileId = new ObjectId();

// Permet d'utiliser l'ID avant l'insertion
const user = {
  _id: userId,
  name: "Alice",
  profileId: profileId  // Référence au profil
};

const profile = {
  _id: profileId,
  userId: userId,       // Référence bidirectionnelle
  bio: "Software Engineer"
};

// Insertion avec références cohérentes
await db.users.insertOne(user);
await db.profiles.insertOne(profile);

// ID disponible immédiatement pour logging
logger.info('User created', { userId: userId.toString() });
```

**Cas d'usage** :
- Relations bidirectionnelles
- Logging avant insertion
- Génération de tokens/URLs avant enregistrement
- Tests unitaires (IDs déterministes)
- Insertions batch avec références

---

## ❌ DON'T : Réutiliser des _id Supprimés

**Explication** : Même après suppression d'un document, son `_id` ne doit jamais être réutilisé.

**Anti-pattern** :
```javascript
// ❌ DANGER : Réutilisation d'_id
const deletedUserId = "user-12345";
await db.users.deleteOne({ _id: deletedUserId });

// Plus tard... ERREUR!
await db.users.insertOne({
  _id: deletedUserId,  // Réutilise l'ancien ID
  name: "New User"
});
```

**Conséquences** :
- **Références orphelines** : D'autres documents peuvent encore référencer l'ancien ID
- **Logs corrompus** : Confusion dans les historiques et audits
- **Caches invalidés** : Les systèmes de cache peuvent avoir l'ancienne valeur
- **Violation de l'intégrité** : L'historique devient incohérent

**Exemple de problème** :
```javascript
// Document A référence userId
const orderHistory = {
  _id: ObjectId("..."),
  userId: "user-12345",  // Référence à l'ancien utilisateur
  orders: [...]
};

// Si on réutilise "user-12345" pour un nouvel utilisateur,
// ce nouvel utilisateur "hérite" de l'historique de l'ancien!
```

**Solution - Soft Delete** :
```javascript
// ✅ Soft delete au lieu de suppression réelle
await db.users.updateOne(
  { _id: userId },
  {
    $set: {
      isDeleted: true,
      deletedAt: new Date()
    }
  }
);

// L'_id n'est jamais réutilisé
// Les références restent cohérentes
// L'historique est préservé
```

---

## ✅ DO : Valider le Format des _id en Entrée

**Explication** : Toute donnée provenant de l'utilisateur, incluant les `_id`, doit être validée rigoureusement.

**Validation robuste** :
```javascript
// ✅ Validation stricte des ObjectId
import { ObjectId } from 'mongodb';

function validateObjectId(id) {
  // Vérifie que c'est une string valide
  if (typeof id !== 'string') {
    throw new Error('ID must be a string');
  }

  // Vérifie le format ObjectId
  if (!ObjectId.isValid(id)) {
    throw new Error('Invalid ObjectId format');
  }

  // Vérifie que la conversion fonctionne
  try {
    return new ObjectId(id);
  } catch (error) {
    throw new Error('Cannot convert to ObjectId');
  }
}

// Usage dans une route API
app.get('/users/:id', async (req, res) => {
  try {
    const userId = validateObjectId(req.params.id);
    const user = await db.users.findOne({ _id: userId });

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    return res.status(400).json({ error: error.message });
  }
});
```

**Validation pour d'autres types** :
```javascript
// ✅ Validation UUID
function validateUUID(id) {
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
  if (!uuidRegex.test(id)) {
    throw new Error('Invalid UUID format');
  }
  return id;
}

// ✅ Validation email (si utilisé comme _id)
function validateEmailId(email) {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    throw new Error('Invalid email format');
  }
  // Normaliser (lowercase)
  return email.toLowerCase().trim();
}
```

---

## ❌ DON'T : Faire Confiance aux _id Utilisateur Sans Validation

**Explication** : Les `_id` non validés peuvent causer des crashs, des failles de sécurité ou des bugs subtils.

**Vulnérabilités** :
```javascript
// ❌ DANGEREUX : Pas de validation
app.get('/users/:id', async (req, res) => {
  // Si req.params.id = "invalid", MongoDB lance une exception
  const user = await db.users.findOne({
    _id: new ObjectId(req.params.id)  // CRASH si format invalide
  });
  res.json(user);
});

// ❌ Injection possible
app.get('/users/:id', async (req, res) => {
  // Si req.params.id = { $gt: "" }, peut retourner tous les users!
  const user = await db.users.findOne({
    _id: req.params.id  // Accepte n'importe quoi
  });
  res.json(user);
});
```

**Conséquences** :
- Crash de l'application
- Exposition de données (injection NoSQL)
- Logs pollués par des erreurs
- Expérience utilisateur dégradée

---

## ✅ DO : Utiliser des _id Composites pour les Relations M2M

**Explication** : Pour les tables de jointure (relations many-to-many), un `_id` composite peut simplifier les requêtes et garantir l'unicité.

**Pattern recommandé** :
```javascript
// ✅ _id composite pour éviter les doublons
// Collection: userGroups (relation many-to-many)
await db.userGroups.insertOne({
  _id: `${userId}_${groupId}`,  // Composite key
  userId: userId,
  groupId: groupId,
  joinedAt: new Date(),
  role: "member"
});

// Garantit l'unicité : un user ne peut rejoindre un groupe qu'une fois
// Requête simple pour vérifier l'appartenance
const membership = await db.userGroups.findOne({
  _id: `${userId}_${groupId}`
});
```

**Avantages** :
- Unicité garantie par construction
- Pas besoin d'index composé unique supplémentaire
- Requêtes simplifiées
- Performance optimale

**Alternative avec objet** :
```javascript
// ✅ _id comme objet composite (plus typé)
await db.userGroups.insertOne({
  _id: {
    userId: new ObjectId(userId),
    groupId: new ObjectId(groupId)
  },
  joinedAt: new Date(),
  role: "member"
});

// Requête
const membership = await db.userGroups.findOne({
  "_id.userId": new ObjectId(userId),
  "_id.groupId": new ObjectId(groupId)
});
```

---

## ❌ DON'T : Créer des _id Trop Complexes

**Explication** : Des `_id` trop complexes deviennent difficiles à manipuler et peuvent causer des problèmes de performance.

**Anti-patterns** :
```javascript
// ❌ _id trop complexe
await db.records.insertOne({
  _id: {
    country: "FR",
    region: "Île-de-France",
    city: "Paris",
    district: "15e",
    street: "Rue de Vaugirard",
    number: "123",
    floor: 4,
    apartment: "B"
  },
  // ...
});

// Requêtes deviennent lourdes
const record = await db.records.findOne({
  "_id.country": "FR",
  "_id.region": "Île-de-France",
  "_id.city": "Paris",
  "_id.district": "15e",
  "_id.street": "Rue de Vaugirard",
  "_id.number": "123",
  "_id.floor": 4,
  "_id.apartment": "B"
});
```

**Conséquences** :
- Requêtes verbeuses et error-prone
- Difficile à indexer efficacement
- Sérialisation/désérialisation coûteuse
- Maintenance complexe

**Solution** :
```javascript
// ✅ _id simple + index composé sur les champs métier
await db.records.insertOne({
  _id: ObjectId("..."),
  address: {
    country: "FR",
    region: "Île-de-France",
    city: "Paris",
    street: "Rue de Vaugirard",
    number: "123",
    floor: 4,
    apartment: "B"
  }
});

// Index composé pour les recherches
db.records.createIndex({
  "address.country": 1,
  "address.city": 1,
  "address.street": 1,
  "address.number": 1
});
```

---

## ✅ DO : Documenter votre Stratégie d'_id

**Explication** : La stratégie de génération des `_id` doit être documentée clairement pour toute l'équipe.

**Documentation recommandée** :
```javascript
/**
 * STRATÉGIE DES IDENTIFIANTS
 *
 * Collections principales (users, orders, products) :
 * - Type : ObjectId (MongoDB default)
 * - Génération : Automatique par MongoDB
 * - Raison : Unicité distribuée, timestamp embarqué
 *
 * Collections de sessions :
 * - Type : UUID v4 (Binary)
 * - Génération : Côté application
 * - Raison : Non-prédictibilité pour sécurité
 *
 * Collections de jointure (userGroups) :
 * - Type : String composite "userId_groupId"
 * - Génération : Côté application
 * - Raison : Garantir l'unicité des paires
 *
 * Collections d'intégration externe (externalProducts) :
 * - Type : String (SKU externe)
 * - Génération : Provient du système externe
 * - Raison : Compatibilité et synchronisation
 */

// Exemple dans le code
const COLLECTIONS_ID_STRATEGY = {
  users: { type: 'ObjectId', generator: 'auto' },
  sessions: { type: 'UUID', generator: 'app' },
  userGroups: { type: 'composite', generator: 'app' }
};
```

---

## Considérations de Performance

### ✅ DO : Considérer l'Impact sur le Sharding

**Explication** : Le choix de l'`_id` est crucial si vous utilisez ou prévoyez d'utiliser le sharding.

**Shard key considerations** :
```javascript
// ❌ Mauvais : Monotone (hot shard)
sh.shardCollection("db.orders", { _id: 1 })
// Tous les nouveaux documents vont sur le même shard

// ✅ Bon : Hashed (distribution uniforme)
sh.shardCollection("db.orders", { _id: "hashed" })
// Distribution équitable entre shards

// ✅ Optimal : Compound shard key
sh.shardCollection("db.orders", { customerId: 1, _id: 1 })
// Distribution par customer + unicité avec _id
```

**Règle** : Si vous prévoyez de sharder sur `_id`, utilisez :
- `_id: "hashed"` pour distribution uniforme
- Un préfixe non-monotone dans un compound shard key

---

### ✅ DO : Considérer la Taille des _id dans les Index

**Explication** : L'`_id` est inclus dans tous les index. Un `_id` volumineux multiplie l'utilisation de la mémoire.

**Impact mesuré** :
```javascript
// ObjectId : 12 bytes
_id: ObjectId("507f1f77bcf86cd799439011")

// UUID string : 36 bytes (3x plus gros)
_id: "550e8400-e29b-41d4-a716-446655440000"

// UUID Binary : 16 bytes (33% plus gros que ObjectId)
_id: Binary(UUID)

// String longue : peut être 100+ bytes
_id: "user_2024-01-15_alice_smith_france_paris_12345"
```

**Calcul d'impact** :
```
Collection : 10 millions de documents
Index : 5 index par collection

ObjectId (12 bytes) :
- Index _id : 120 MB
- Tous les index incluent _id : +60 MB par index
- Total : 120 + (5 × 60) = 420 MB

UUID String (36 bytes) :
- Index _id : 360 MB
- Tous les index : +180 MB par index
- Total : 360 + (5 × 180) = 1.26 GB

= 3x plus d'espace mémoire requis
```

**Recommandation** : Privilégier les types compacts (ObjectId, UUID Binary).

---

## Cas d'Usage Spécifiques

### ✅ DO : Utiliser ULID pour Ordre Temporel + Aléatoire

**Explication** : ULID (Universally Unique Lexicographically Sortable Identifier) combine les avantages d'ObjectId et UUID.

**Caractéristiques** :
```javascript
// ✅ ULID : 01ARZ3NDEKTSV4RRFFQ69G5FAV
import { ulid } from 'ulid';

const userId = ulid();
await db.users.insertOne({
  _id: userId,  // Sortable + random
  name: "Alice"
});
```

**Avantages** :
- **Tri chronologique** : Comme ObjectId
- **Non-prédictible** : Composant aléatoire de 80 bits
- **Lexicographiquement sortable** : Peut être trié comme string
- **Compact** : 26 caractères (vs 36 pour UUID)
- **Lisible** : Base32 (vs hex pour UUID)

**Quand utiliser** :
- APIs publiques nécessitant ordre temporel
- Besoin de non-prédictibilité ET tri chronologique
- Identifiants exposés dans les URLs

---

### ✅ DO : Utiliser Snowflake IDs pour Très Haute Performance

**Explication** : Les Snowflake IDs (popularisés par Twitter) offrent performance maximale dans des systèmes distribués haute fréquence.

**Structure** :
```
64 bits total :
- 41 bits : Timestamp milliseconde
- 10 bits : Machine ID
- 12 bits : Sequence number

= 4096 IDs/ms par machine
= ~4 millions d'IDs/seconde par machine
```

**Implémentation** :
```javascript
// ✅ Snowflake ID (exemple simplifié)
class SnowflakeId {
  constructor(machineId) {
    this.machineId = machineId & 0x3FF; // 10 bits
    this.sequence = 0;
    this.lastTimestamp = 0;
  }

  generate() {
    let timestamp = Date.now();

    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1) & 0xFFF; // 12 bits
      if (this.sequence === 0) {
        // Wait for next millisecond
        while (timestamp <= this.lastTimestamp) {
          timestamp = Date.now();
        }
      }
    } else {
      this.sequence = 0;
    }

    this.lastTimestamp = timestamp;

    // Combine timestamp, machine, sequence
    const id = (timestamp << 22) | (this.machineId << 12) | this.sequence;
    return id;
  }
}

const idGenerator = new SnowflakeId(1); // Machine 1

await db.events.insertOne({
  _id: idGenerator.generate(),  // Numeric ID
  eventType: "click",
  timestamp: new Date()
});
```

**Quand utiliser** :
- Très haute fréquence d'insertion (>100k/sec)
- Systèmes distribués avec coordination minimale
- Besoin de IDs entiers (compatibilité legacy)
- IoT, analytics, logging haute fréquence

**Attention** : Complexité de gestion (synchronisation horloges, attribution machine IDs).

---

## Migrations et Évolution

### ✅ DO : Planifier les Migrations d'_id

**Explication** : Si vous devez changer de stratégie d'`_id`, planifiez une migration progressive.

**Stratégie de migration** :
```javascript
// ✅ Migration progressive avec double écriture

// Phase 1 : Ajouter nouveau champ
await db.users.updateMany(
  { newId: { $exists: false } },
  [{ $set: { newId: { $function: {
    body: function() { return new ObjectId(); },
    lang: "js"
  }}}}]
);

// Phase 2 : Double écriture (nouveau code)
const newUserId = new ObjectId();
await db.users.insertOne({
  _id: oldStyleId,       // Ancien format
  newId: newUserId,      // Nouveau format
  // ...
});

// Phase 3 : Migration des références
// Mettre à jour toutes les collections référençant l'ancien _id

// Phase 4 : Switch (nouveau code utilise newId)

// Phase 5 : Cleanup (supprimer anciens _id)
// Créer nouvelle collection avec newId comme _id
// Migrer les données
// Supprimer ancienne collection
```

---

### ❌ DON'T : Migrer les _id Sans Plan de Rollback

**Explication** : Une migration d'`_id` mal préparée peut détruire l'intégrité de toute la base.

**Checklist avant migration** :
- [ ] Backup complet de la base
- [ ] Test de migration sur environnement de staging
- [ ] Plan de rollback documenté et testé
- [ ] Fenêtre de maintenance planifiée
- [ ] Monitoring des références cassées
- [ ] Validation de l'intégrité post-migration
- [ ] Communication aux équipes

**Temps estimé** : Pour 100M de documents, prévoir plusieurs heures à plusieurs jours selon la complexité des références.

---

## Sécurité

### ✅ DO : Ne Pas Exposer les ObjectIds Séquentiels

**Explication** : Même si ObjectIds ne sont pas strictement séquentiels, le composant timestamp peut révéler des informations.

**Risques** :
```javascript
// ObjectId : 507f1f77bcf86cd799439011
//           ^^^^^^^^ = timestamp
// On peut déduire la date de création

// Si exposé dans URL publique
GET /api/orders/507f1f77bcf86cd799439011

// Attaquant peut :
// 1. Estimer le volume (tentatives d'énumération)
// 2. Deviner la date de création
// 3. Essayer des ObjectIds adjacents
```

**Solutions** :
```javascript
// ✅ Option 1 : UUID pour les ressources publiques
{
  _id: ObjectId("..."),              // ID interne
  publicId: "550e8400-e29b-...",     // ID public (UUID)
  orderNumber: "ORD-2024-00123"      // Numéro métier
}

// ✅ Option 2 : Tokens signés
const publicToken = jwt.sign(
  { orderId: objectId.toString() },
  SECRET_KEY,
  { expiresIn: '24h' }
);

// URL : /api/orders/eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

### ❌ DON'T : Utiliser _id Prédictibles pour Ressources Sensibles

**Explication** : Les `_id` facilement devinables facilitent les attaques par énumération.

**Scénario d'attaque** :
```javascript
// ❌ IDs séquentiels pour documents privés
{
  _id: 1,
  userId: "alice",
  privateDocument: "Confidential data"
}

// Attaquant peut énumérer :
for (let i = 1; i < 100000; i++) {
  fetch(`/api/documents/${i}`)
  // Tester tous les documents même sans autorisation
}
```

**Protection** :
```javascript
// ✅ IDs non-prédictibles
{
  _id: "550e8400-e29b-41d4-a716-446655440000",
  userId: "alice",
  privateDocument: "Confidential data"
}

// + Vérification d'autorisation côté serveur
app.get('/api/documents/:id', auth, async (req, res) => {
  const doc = await db.documents.findOne({ _id: req.params.id });

  // Vérifier que l'utilisateur a le droit d'accès
  if (doc.userId !== req.user.id) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  res.json(doc);
});
```

---

## Checklist Gestion des _id

### Choix du Type
- [ ] Raison valable de ne pas utiliser ObjectId documentée
- [ ] Type choisi adapté au cas d'usage
- [ ] Impact sur performance et stockage évalué
- [ ] Compatibilité avec sharding vérifiée

### Génération
- [ ] Stratégie de génération documentée
- [ ] Génération distribuée (pas de point de contention)
- [ ] Unicité garantie dans tous les cas
- [ ] Tests de collision effectués

### Validation
- [ ] Validation des _id en entrée utilisateur
- [ ] Gestion d'erreurs appropriée
- [ ] Format vérifié avant requêtes
- [ ] Protection contre injections

### Sécurité
- [ ] Pas d'exposition d'IDs prédictibles
- [ ] Pas de fuite d'informations via _id
- [ ] Autorisations vérifiées indépendamment de l'_id
- [ ] URLs publiques avec IDs aléatoires

### Évolution
- [ ] Pas de données mutables dans _id
- [ ] Plan de migration si changement nécessaire
- [ ] Soft delete au lieu de suppression + réutilisation
- [ ] Références maintenues lors d'évolutions

---

## Tableau Comparatif des Types d'_id

| Type | Taille | Unicité | Ordre | Génération | Usage Recommandé |
|------|--------|---------|-------|------------|------------------|
| **ObjectId** | 12 bytes | ✅ Garantie | ✅ Chronologique | Distribuée | ✅ **Défaut** - Majorité des cas |
| **UUID v4** | 16 bytes | ✅ Garantie | ❌ Aléatoire | Distribuée | APIs publiques, tokens |
| **ULID** | 26 chars | ✅ Garantie | ✅ Chronologique | Distribuée | APIs avec ordre + sécurité |
| **Snowflake** | 8 bytes (int64) | ✅ Garantie | ✅ Chronologique | Coordonnée | Très haute fréquence |
| **Auto-increment** | Variable | ✅ Locale | ✅ Séquentiel | Centralisée | ❌ **Anti-pattern** |
| **String custom** | Variable | ⚠️ À vérifier | ⚠️ Dépend | Application | Identifiants métier naturels |
| **Composite** | Variable | ✅ Par design | ❌ Non | Application | Relations M2M |

---

## Conclusion

La gestion des `_id` est un élément fondamental qui impacte tous les aspects de votre application MongoDB :

- **Performance** : Distribution des écritures, taille des index
- **Scalabilité** : Compatibilité sharding, architecture distribuée
- **Sécurité** : Prédictibilité, exposition d'informations
- **Maintenabilité** : Immutabilité, évolution du système

**Règles d'or** :
1. **ObjectId par défaut** sauf raison valable documentée
2. **Immutabilité** : Jamais de données changeantes dans _id
3. **Validation** : Toujours valider les _id en entrée
4. **Documentation** : Stratégie claire et partagée
5. **Sécurité** : Ne pas exposer d'IDs prédictibles

Un choix judicieux d'`_id` au début du projet évite des migrations coûteuses et complexes plus tard.

---


⏭️ [Gestion des null et valeurs manquantes](/21-bonnes-pratiques-anti-patterns/03-gestion-null-valeurs-manquantes.md)
