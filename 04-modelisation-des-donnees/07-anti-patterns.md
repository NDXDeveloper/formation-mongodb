🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.7 Anti-patterns à éviter

## Introduction

Un **anti-pattern** est une solution apparemment logique à un problème, mais qui s'avère en réalité **inefficace, contre-productive ou dangereuse** dans la pratique. En modélisation MongoDB, les anti-patterns résultent souvent d'habitudes issues du monde relationnel (SQL) ou d'une méconnaissance des spécificités de MongoDB.

Connaître et éviter ces anti-patterns est **aussi important** que de maîtriser les bonnes pratiques. Dans ce chapitre, nous allons explorer les erreurs les plus courantes, comprendre pourquoi elles posent problème et apprendre comment les corriger.

---

## Pourquoi éviter les anti-patterns ?

### Conséquences des anti-patterns

Les anti-patterns peuvent entraîner :

- 🔴 **Performances dégradées** : requêtes lentes, temps de réponse élevés
- 🔴 **Problèmes de scalabilité** : impossible de gérer la croissance des données
- 🔴 **Maintenance complexe** : code difficile à modifier et à faire évoluer
- 🔴 **Incohérences de données** : risques de corruption ou de perte de données
- 🔴 **Coûts élevés** : infrastructure surdimensionnée pour compenser les inefficacités
- 🔴 **Bugs difficiles à tracer** : comportements imprévisibles en production

### Principe général

La plupart des anti-patterns MongoDB viennent de :

1. **Penser relationnel** : appliquer aveuglément les principes SQL
2. **Ignorer les limites** : ne pas tenir compte des contraintes de MongoDB
3. **Ne pas mesurer** : optimiser sans données réelles
4. **Négliger la croissance** : ne pas anticiper l'évolution du volume

---

## Anti-pattern 1 : Normalisation excessive

### ❌ Le problème

Normaliser systématiquement comme en SQL en créant de nombreuses collections avec des références partout.

**Exemple d'anti-pattern :**

```javascript
// Collection "utilisateurs"
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  nom: "Dupont",
  prenom: "Sophie",
  emailId: ObjectId("...")  // ← Référence pour un simple email !
}

// Collection "emails" (inutile !)
{
  _id: ObjectId("..."),
  adresse: "sophie.dupont@example.com"
}

// Collection "telephones" (inutile !)
{
  _id: ObjectId("..."),
  numero: "+33 6 12 34 56 78"
}

// Collection "adresses"
{
  _id: ObjectId("..."),
  rueId: ObjectId("..."),  // ← Encore une référence !
  villeId: ObjectId("..."),
  codePostalId: ObjectId("...")
}
```

### 🔴 Pourquoi c'est problématique

- **Requêtes multiples** : besoin de 4-5 requêtes pour afficher un profil utilisateur complet
- **Performance catastrophique** : latence multipliée par le nombre de jointures
- **Complexité inutile** : code difficile à maintenir
- **Overhead** : beaucoup plus de documents à gérer
- **Pas d'atomicité** : impossible de mettre à jour de manière cohérente

### ✅ La bonne approche

Imbriquer les données qui sont **toujours consultées ensemble** :

```javascript
// Collection "utilisateurs" (tout en un)
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  nom: "Dupont",
  prenom: "Sophie",
  email: "sophie.dupont@example.com",  // ← Directement dans le document
  telephone: "+33 6 12 34 56 78",      // ← Pas besoin de collection séparée
  adresse: {                            // ← Imbriqué
    rue: "12 rue de la République",
    ville: "Lyon",
    codePostal: "69001",
    pays: "France"
  },
  dateInscription: ISODate("2024-01-10")
}
```

**Avantages :**
- ✅ Une seule requête pour tout
- ✅ Performance optimale
- ✅ Atomicité garantie
- ✅ Code simple et clair

### 📊 Impact mesuré

```javascript
// ❌ Anti-pattern : 5 requêtes
Temps total : ~50-100ms (5 × 10-20ms par requête)

// ✅ Bonne pratique : 1 requête
Temps total : ~10-20ms
Gain : 3-5x plus rapide
```

---

## Anti-pattern 2 : Documents trop volumineux

### ❌ Le problème

Imbriquer des milliers d'éléments dans un seul document sans considérer la limite de 16 Mo.

**Exemple d'anti-pattern :**

```javascript
// Article de blog viral avec TOUS les commentaires imbriqués
{
  _id: ObjectId("..."),
  titre: "Article viral",
  contenu: "...",
  commentaires: [
    // ⚠️ 50 000 commentaires imbriqués !
    { auteur: "User1", texte: "...", date: ISODate("...") },
    { auteur: "User2", texte: "...", date: ISODate("...") },
    // ... 49 998 autres commentaires
  ]
}
```

### 🔴 Pourquoi c'est problématique

- **Dépassement de la limite** : risque d'atteindre 16 Mo et de ne plus pouvoir écrire
- **Performance dégradée** : charger un document de 15 Mo à chaque requête
- **Mémoire** : consommation excessive de RAM
- **Pagination impossible** : difficile d'afficher les commentaires par pages
- **Modification lente** : ajouter un commentaire nécessite de réécrire tout le document

### ✅ La bonne approche

Utiliser des **références** pour les relations one-to-many avec beaucoup d'éléments :

```javascript
// Collection "articles"
{
  _id: ObjectId("..."),
  titre: "Article viral",
  contenu: "...",
  nombreCommentaires: 50000,  // ← Compteur pour affichage
  datePublication: ISODate("...")
}

// Collection "commentaires" (séparée)
{
  _id: ObjectId("..."),
  articleId: ObjectId("..."),  // ← Référence
  auteur: "User1",
  texte: "Super article !",
  date: ISODate("..."),
  likes: 15
}
```

**Requête optimisée avec pagination :**

```javascript
// Afficher l'article
const article = db.articles.findOne({ _id: articleId })

// Charger les 20 premiers commentaires
const commentaires = db.commentaires
  .find({ articleId: article._id })
  .sort({ date: -1 })
  .limit(20)
  .skip(0)
```

### 💡 Alternative : Pattern Subset

Combiner les deux approches pour le meilleur compromis :

```javascript
{
  _id: ObjectId("..."),
  titre: "Article viral",
  contenu: "...",
  commentairesRecents: [  // ← Subset : 5 derniers commentaires
    { auteur: "User1", texte: "...", date: ISODate("...") },
    { auteur: "User2", texte: "...", date: ISODate("...") }
  ],
  nombreCommentairesTotal: 50000,
  datePublication: ISODate("...")
}
```

---

## Anti-pattern 3 : Tableaux illimités

### ❌ Le problème

Créer des tableaux qui peuvent **croître indéfiniment** sans limite.

**Exemple d'anti-pattern :**

```javascript
// Réseau social : tous les posts dans le profil
{
  _id: ObjectId("..."),
  nom: "Sophie Martin",
  posts: [
    // ⚠️ Utilisateur actif depuis 5 ans = 10 000+ posts !
    { texte: "Mon premier post", date: ISODate("2019-01-01") },
    { texte: "Deuxième post", date: ISODate("2019-01-02") },
    // ... 9 998 autres posts
  ]
}

// E-commerce : tous les achats dans le profil client
{
  _id: ObjectId("..."),
  nom: "Jean Dupont",
  achats: [
    // ⚠️ Client fidèle = 1 000+ commandes !
  ]
}

// Application de messagerie : tous les messages dans la conversation
{
  _id: ObjectId("..."),
  participants: ["user1", "user2"],
  messages: [
    // ⚠️ Conversation active depuis des années = 100 000+ messages !
  ]
}
```

### 🔴 Pourquoi c'est problématique

- **Dépassement de 16 Mo** : garantie de problème à moyen terme
- **Performance** : chargement de plus en plus lent au fil du temps
- **Index** : impossible d'indexer efficacement un tableau géant
- **$push devient lent** : ajouter un élément nécessite de réécrire tout le document

### ✅ La bonne approche

**Solution 1 : Collection séparée avec référence**

```javascript
// Collection "utilisateurs"
{
  _id: ObjectId("..."),
  nom: "Sophie Martin",
  email: "sophie@example.com",
  statistiques: {
    nombrePosts: 10234  // ← Compteur dénormalisé
  }
}

// Collection "posts" (séparée)
{
  _id: ObjectId("..."),
  auteurId: ObjectId("..."),
  texte: "Mon post",
  date: ISODate("..."),
  likes: 42
}

// Index pour performance
db.posts.createIndex({ auteurId: 1, date: -1 })
```

**Solution 2 : Bucketing (regroupement)**

```javascript
// Regrouper les posts par période (1 mois)
{
  _id: ObjectId("..."),
  auteurId: ObjectId("..."),
  annee: 2024,
  mois: 1,
  posts: [
    // ← Maximum ~100 posts/mois (raisonnable)
    { texte: "...", date: ISODate("2024-01-15") },
    { texte: "...", date: ISODate("2024-01-16") }
  ],
  nombrePosts: 87
}
```

---

## Anti-pattern 4 : Pas de computed fields

### ❌ Le problème

Calculer des valeurs à chaque requête au lieu de les précalculer.

**Exemple d'anti-pattern :**

```javascript
// Commande sans total précalculé
{
  _id: ObjectId("..."),
  numeroCommande: "CMD-001",
  articles: [
    { nom: "Produit A", quantite: 2, prixUnitaire: 29.99 },
    { nom: "Produit B", quantite: 1, prixUnitaire: 89.99 },
    { nom: "Produit C", quantite: 3, prixUnitaire: 19.99 }
  ]
  // ❌ Pas de total : il faut calculer à chaque fois !
}

// Application le calcule à chaque affichage
const total = commande.articles.reduce((sum, article) => {
  return sum + (article.quantite * article.prixUnitaire)
}, 0)
// ⚠️ Calcul répété des millions de fois !
```

**Article sans statistiques précalculées :**

```javascript
{
  _id: ObjectId("..."),
  titre: "Article",
  contenu: "..."
  // ❌ Pas de compteurs
}

// Pour afficher le nombre de likes, il faut compter à chaque fois
const nombreLikes = db.likes.countDocuments({ articleId: article._id })
// ⚠️ Requête supplémentaire à chaque affichage !
```

### 🔴 Pourquoi c'est problématique

- **Performance** : calculs répétés inutilement
- **Latence** : temps de réponse accru
- **CPU** : charge serveur inutile
- **Requêtes supplémentaires** : pour obtenir les compteurs

### ✅ La bonne approche

**Précalculer lors de l'écriture** (Pattern Computed) :

```javascript
// Commande avec total précalculé
{
  _id: ObjectId("..."),
  numeroCommande: "CMD-001",
  articles: [
    { nom: "Produit A", quantite: 2, prixUnitaire: 29.99, sousTotal: 59.98 },
    { nom: "Produit B", quantite: 1, prixUnitaire: 89.99, sousTotal: 89.99 },
    { nom: "Produit C", quantite: 3, prixUnitaire: 19.99, sousTotal: 59.97 }
  ],
  sousTotal: 209.94,        // ← Précalculé
  tva: 41.99,               // ← Précalculé
  fraisPort: 5.00,          // ← Précalculé
  total: 256.93             // ← Précalculé
}

// Article avec statistiques
{
  _id: ObjectId("..."),
  titre: "Article",
  contenu: "...",
  statistiques: {           // ← Précalculées
    vues: 5234,
    likes: 342,
    commentaires: 89,
    partages: 23
  },
  derniereMiseAJourStats: ISODate("2024-01-20T10:00:00Z")
}
```

**Mise à jour des compteurs :**

```javascript
// Incrémenter le compteur lors d'un like
db.articles.updateOne(
  { _id: articleId },
  {
    $inc: { "statistiques.likes": 1 },
    $set: { derniereMiseAJourStats: new Date() }
  }
)
```

### 📊 Impact

```
❌ Sans computed : Calcul à chaque affichage = 1-5ms × nombre de vues
✅ Avec computed : Lecture directe = 0.1ms

Pour un article avec 10 000 vues/jour :
❌ Sans : 10 000 × 3ms = 30 secondes de CPU/jour
✅ Avec : 10 000 × 0.1ms = 1 seconde de CPU/jour
Gain : 30x moins de charge serveur
```

---

## Anti-pattern 5 : Mauvaise gestion des références

### ❌ Le problème

**Problème A : Références orphelines**

Supprimer un document sans nettoyer les références vers lui.

```javascript
// Supprimer un auteur
db.auteurs.deleteOne({ _id: auteurId })

// ❌ Les articles gardent la référence vers un auteur qui n'existe plus !
{
  _id: ObjectId("..."),
  titre: "Article",
  auteurId: ObjectId("..."),  // ← Référence morte !
  contenu: "..."
}
```

**Problème B : Références bidirectionnelles incohérentes**

```javascript
// Étudiant liste le cours
{
  _id: ObjectId("etudiant1"),
  nom: "Sophie",
  coursIds: [ObjectId("cours1"), ObjectId("cours2")]
}

// ❌ Mais le cours ne liste pas l'étudiant !
{
  _id: ObjectId("cours1"),
  titre: "MongoDB",
  etudiantIds: [ObjectId("etudiant2"), ObjectId("etudiant3")]
  // ← Sophie manque !
}
```

### 🔴 Pourquoi c'est problématique

- **Incohérence** : données désynchronisées
- **Erreurs applicatives** : références vers des documents inexistants
- **Difficile à debugger** : bugs intermittents et imprévisibles
- **Nettoyage complexe** : nécessite des scripts de maintenance

### ✅ La bonne approche

**Solution 1 : Utiliser des transactions pour les références bidirectionnelles**

```javascript
const session = db.getMongo().startSession()
session.startTransaction()

try {
  // Inscrire étudiant au cours
  db.etudiants.updateOne(
    { _id: etudiantId },
    { $addToSet: { coursIds: coursId } },
    { session }
  )

  db.cours.updateOne(
    { _id: coursId },
    { $addToSet: { etudiantIds: etudiantId } },
    { session }
  )

  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}
```

**Solution 2 : Collection de jonction (recommandé)**

```javascript
// Éviter les références bidirectionnelles, utiliser une jonction
// Collection "inscriptions"
{
  _id: ObjectId("..."),
  etudiantId: ObjectId("..."),
  coursId: ObjectId("..."),
  dateInscription: ISODate("..."),
  statut: "active"
}

// Index pour éviter doublons
db.inscriptions.createIndex(
  { etudiantId: 1, coursId: 1 },
  { unique: true }
)
```

**Solution 3 : Suppression en cascade**

```javascript
// Supprimer un auteur ET nettoyer les références
const session = db.getMongo().startSession()
session.startTransaction()

try {
  // Supprimer l'auteur
  db.auteurs.deleteOne({ _id: auteurId }, { session })

  // Nettoyer les références dans les articles
  db.articles.updateMany(
    { auteurId: auteurId },
    {
      $unset: { auteurId: "" },
      $set: { auteurSupprime: true }
    },
    { session }
  )

  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}
```

---

## Anti-pattern 6 : Ignorer les index

### ❌ Le problème

Ne pas créer d'index sur les champs fréquemment utilisés pour les requêtes.

**Exemple d'anti-pattern :**

```javascript
// ❌ Aucun index créé
db.produits.find({ categorieId: ObjectId("...") })
// ⚠️ Collection scan : parcourt TOUS les documents !

db.utilisateurs.find({ email: "sophie@example.com" })
// ⚠️ Collection scan sur 1 million d'utilisateurs !

db.commandes.find({
  statut: "en_preparation",
  dateCommande: { $gte: ISODate("2024-01-01") }
})
// ⚠️ Scan complet à chaque requête !
```

### 🔴 Pourquoi c'est problématique

- **Performance catastrophique** : requêtes qui prennent des secondes au lieu de millisecondes
- **Scalabilité** : impossible de gérer la croissance des données
- **CPU élevé** : serveur surchargé
- **Expérience utilisateur** : application lente et peu réactive

### ✅ La bonne approche

**Créer des index sur les champs de requête :**

```javascript
// Index sur categorieId
db.produits.createIndex({ categorieId: 1 })

// Index unique sur email
db.utilisateurs.createIndex({ email: 1 }, { unique: true })

// Index composé
db.commandes.createIndex({ statut: 1, dateCommande: -1 })

// Vérifier avec explain()
db.produits.find({ categorieId: ObjectId("...") }).explain("executionStats")
```

### 📊 Impact des index

```javascript
// Sans index
{
  executionTimeMillis: 1523,
  totalDocsExamined: 1000000,
  totalKeysExamined: 0,
  stage: "COLLSCAN"  // ← Collection scan = mauvais !
}

// Avec index
{
  executionTimeMillis: 5,
  totalDocsExamined: 123,
  totalKeysExamined: 123,
  stage: "IXSCAN"  // ← Index scan = bon !
}

Gain : 300x plus rapide
```

---

## Anti-pattern 7 : Utiliser MongoDB comme un cache

### ❌ Le problème

Stocker des données temporaires ou des sessions dans MongoDB au lieu d'utiliser un vrai cache.

**Exemple d'anti-pattern :**

```javascript
// ❌ Stocker les sessions utilisateurs dans MongoDB
{
  _id: ObjectId("..."),
  sessionId: "abc123xyz",
  userId: ObjectId("..."),
  data: { /* données session */ },
  expireAt: ISODate("2024-01-20T10:30:00Z")
}

// ❌ Cache de résultats de calculs
{
  _id: ObjectId("..."),
  cacheKey: "stats-user-123",
  value: { /* résultat */ },
  ttl: 300
}
```

### 🔴 Pourquoi c'est problématique

- **Performance sous-optimale** : MongoDB n'est pas optimisé pour le caching
- **Latence plus élevée** : accès disque vs RAM pure
- **Overhead** : journalisation, réplication inutile pour du cache
- **Coût** : utilisation de ressources précieuses pour du temporaire

### ✅ La bonne approche

**Utiliser Redis ou Memcached pour le cache :**

```javascript
// ✅ Redis pour les sessions
await redis.set(
  `session:${sessionId}`,
  JSON.stringify(sessionData),
  'EX',
  3600  // 1 heure
)

// ✅ Redis pour le cache
await redis.set(
  `cache:stats:${userId}`,
  JSON.stringify(stats),
  'EX',
  300  // 5 minutes
)
```

**MongoDB pour les données persistantes :**

```javascript
// ✅ MongoDB pour les données métier
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  preferences: { /* préférences permanentes */ },
  historique: [ /* actions importantes */ ]
}
```

**Exception : TTL Index pour le cleanup automatique**

```javascript
// OK : Utiliser TTL index pour nettoyage automatique
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })

// Document se supprime automatiquement après 24h
{
  _id: ObjectId("..."),
  message: "Log entry",
  createdAt: ISODate("2024-01-20T10:00:00Z")
}
```

---

## Anti-pattern 8 : Schéma trop flexible

### ❌ Le problème

Abuser de la flexibilité de MongoDB en n'ayant aucune structure cohérente.

**Exemple d'anti-pattern :**

```javascript
// Document 1
{
  _id: ObjectId("..."),
  nom: "Dupont",
  prenom: "Sophie"
}

// Document 2 (structure différente)
{
  _id: ObjectId("..."),
  name: "Martin Jean",  // ← Nom de champ différent !
  email: "martin@example.com"
}

// Document 3 (encore différent)
{
  _id: ObjectId("..."),
  fullName: { first: "Pierre", last: "Leclerc" },  // ← Structure imbriquée
  mail: "pierre@example.com"  // ← "mail" au lieu de "email"
}

// Document 4 (types incohérents)
{
  _id: ObjectId("..."),
  nom: "Durand",
  age: "28"  // ← String au lieu de Number
}
```

### 🔴 Pourquoi c'est problématique

- **Code complexe** : besoin de gérer tous les cas possibles
- **Bugs** : erreurs difficiles à prévoir
- **Requêtes difficiles** : impossible de chercher efficacement
- **Index inefficaces** : index qui ne couvrent pas tous les cas
- **Maintenance cauchemardesque** : évolution impossible

### ✅ La bonne approche

**Définir une structure cohérente avec validation :**

```javascript
// ✅ Structure cohérente
{
  _id: ObjectId("..."),
  nom: "Dupont",
  prenom: "Sophie",
  email: "sophie@example.com",
  age: 28,  // ← Toujours Number
  dateInscription: ISODate("2024-01-10")
}

// Validation de schéma
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prenom", "email"],
      properties: {
        nom: {
          bsonType: "string",
          description: "Nom de famille requis"
        },
        prenom: {
          bsonType: "string",
          description: "Prénom requis"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Email valide requis"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150,
          description: "Âge doit être un nombre entier"
        }
      }
    }
  }
})
```

**Note :** La flexibilité est une force, mais elle doit être **contrôlée** !

---

## Anti-pattern 9 : Duplication sans gestion

### ❌ Le problème

Dénormaliser des données sans stratégie de mise à jour.

**Exemple d'anti-pattern :**

```javascript
// Commande avec infos client dénormalisées
{
  _id: ObjectId("..."),
  numeroCommande: "CMD-001",
  client: {
    nom: "Dupont",
    email: "dupont@example.com",
    telephone: "+33 6 12 34 56 78",
    adresse: "12 rue de la Paix, Paris"
  },
  articles: [ /* ... */ ],
  total: 299.99
}

// ❌ Client change son email
// Il faut mettre à jour TOUTES ses commandes !
db.clients.updateOne({ _id: clientId }, { $set: { email: "nouveau@example.com" } })
// ⚠️ Mais les commandes gardent l'ancien email !
```

### 🔴 Pourquoi c'est problématique

- **Incohérence** : données obsolètes dans plusieurs documents
- **Maintenance complexe** : mises à jour massives nécessaires
- **Risque d'oubli** : certaines copies non mises à jour
- **Performance** : mises à jour coûteuses

### ✅ La bonne approche

**Option 1 : Dénormaliser seulement ce qui ne change jamais**

```javascript
// ✅ Snapshot au moment de la commande (historique)
{
  _id: ObjectId("..."),
  numeroCommande: "CMD-001",
  clientId: ObjectId("..."),  // ← Référence
  clientSnapshot: {  // ← Snapshot historique (ne sera PAS mis à jour)
    nom: "Dupont",
    email: "dupont@example.com",  // Email au moment de la commande
    adresseLivraison: "12 rue de la Paix"
  },
  dateCommande: ISODate("2024-01-15")
}
```

**Option 2 : Dénormaliser avec stratégie de mise à jour**

```javascript
// Dénormalisation avec date de sync
{
  _id: ObjectId("..."),
  titre: "Article",
  auteurId: ObjectId("..."),
  auteurCache: {
    nom: "Jean Martin",
    photo: "https://...",
    dateMiseAJour: ISODate("2024-01-15")
  }
}

// Script de synchronisation périodique
db.articles.updateMany(
  {
    "auteurId": auteurId,
    "auteurCache.dateMiseAJour": { $lt: auteur.dateModification }
  },
  {
    $set: {
      "auteurCache.nom": auteur.nom,
      "auteurCache.photo": auteur.photo,
      "auteurCache.dateMiseAJour": new Date()
    }
  }
)
```

**Option 3 : Change Streams pour synchronisation automatique**

```javascript
// Écouter les changements sur la collection auteurs
const changeStream = db.auteurs.watch()

changeStream.on('change', async (change) => {
  if (change.operationType === 'update') {
    const auteurId = change.documentKey._id

    // Mettre à jour tous les articles de cet auteur
    await db.articles.updateMany(
      { "auteurId": auteurId },
      {
        $set: {
          "auteurCache.nom": change.updateDescription.updatedFields.nom,
          "auteurCache.photo": change.updateDescription.updatedFields.photo
        }
      }
    )
  }
})
```

---

## Anti-pattern 10 : Ne pas utiliser de transactions quand nécessaire

### ❌ Le problème

Effectuer des opérations multi-documents sans transaction, risquant l'incohérence.

**Exemple d'anti-pattern :**

```javascript
// ❌ Transfert d'argent sans transaction
// Étape 1 : Débiter le compte A
db.comptes.updateOne(
  { _id: compteA },
  { $inc: { solde: -100 } }
)

// ⚠️ Si le serveur crash ici, l'argent disparaît !

// Étape 2 : Créditer le compte B
db.comptes.updateOne(
  { _id: compteB },
  { $inc: { solde: 100 } }
)
```

### 🔴 Pourquoi c'est problématique

- **Incohérence** : opérations partiellement exécutées
- **Perte de données** : argent qui "disparaît"
- **Bugs critiques** : erreurs difficiles à tracer
- **Non-atomicité** : pas de garantie ACID

### ✅ La bonne approche

**Utiliser des transactions pour les opérations critiques :**

```javascript
// ✅ Transfert avec transaction
const session = db.getMongo().startSession()
session.startTransaction()

try {
  // Débiter compte A
  db.comptes.updateOne(
    { _id: compteA },
    { $inc: { solde: -100 } },
    { session }
  )

  // Créditer compte B
  db.comptes.updateOne(
    { _id: compteB },
    { $inc: { solde: 100 } },
    { session }
  )

  // Créer un historique
  db.transactions.insertOne({
    type: "transfert",
    source: compteA,
    destination: compteB,
    montant: 100,
    date: new Date()
  }, { session })

  // Tout ou rien
  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}
```

---

## Checklist anti-patterns

### ✅ Vérifications à faire

Avant de déployer votre modèle, vérifiez :

- [ ] **Pas de normalisation excessive** : données liées sont imbriquées quand approprié
- [ ] **Pas de documents > 1 Mo** : vérifier avec `Object.bsonsize(doc)`
- [ ] **Tableaux limités** : aucun tableau ne peut croître indéfiniment
- [ ] **Computed fields** : totaux et statistiques sont précalculés
- [ ] **Références propres** : stratégie pour les orphelins et l'incohérence
- [ ] **Index présents** : tous les champs de requête sont indexés
- [ ] **Pas de cache MongoDB** : utiliser Redis/Memcached pour ça
- [ ] **Schéma cohérent** : validation activée, structure homogène
- [ ] **Duplication gérée** : stratégie de mise à jour ou snapshot historique
- [ ] **Transactions** : utilisées pour opérations multi-documents critiques

### 🔍 Scripts de détection

```javascript
// Détecter les documents volumineux
db.collection.find().forEach(doc => {
  const size = Object.bsonsize(doc)
  if (size > 1024 * 1024) {  // > 1 Mo
    print(`Document ${doc._id} : ${size} bytes`)
  }
})

// Détecter les tableaux larges
db.collection.aggregate([
  {
    $project: {
      tailleTableau: { $size: "$monTableau" }
    }
  },
  { $match: { tailleTableau: { $gt: 100 } } },
  { $sort: { tailleTableau: -1 } }
])

// Vérifier les index manquants
db.collection.find({ categorieId: ObjectId("...") }).explain("executionStats")
// Si COLLSCAN apparaît → index manquant !
```

---

## Conclusion

Les anti-patterns sont des **pièges courants** dans la modélisation MongoDB. En les connaissant, vous pouvez :

- ✅ **Éviter les erreurs** coûteuses dès la conception
- ✅ **Optimiser les performances** de votre application
- ✅ **Faciliter la maintenance** à long terme
- ✅ **Garantir la scalabilité** de votre système

**Règles d'or pour éviter les anti-patterns :**

1. 🎯 **Pensez MongoDB, pas SQL** : profitez de la flexibilité du document
2. 🎯 **Anticipez la croissance** : vos données vont grossir !
3. 🎯 **Mesurez toujours** : utilisez `explain()` et le profiler
4. 🎯 **Validez votre schéma** : structure cohérente même si flexible
5. 🎯 **Testez en conditions réelles** : volume et charge réalistes
6. 🎯 **Documentez vos choix** : expliquez votre modélisation
7. 🎯 **Revoyez régulièrement** : le modèle doit évoluer avec l'application

N'oubliez pas : **il vaut mieux prévenir que guérir**. Prendre le temps de bien modéliser dès le départ vous évitera des refactorings coûteux plus tard !

---

**Points clés à retenir :**

- ✅ La normalisation excessive tue les performances dans MongoDB
- ✅ Attention à la limite de 16 Mo par document
- ✅ Les tableaux illimités sont un piège garanti
- ✅ Précalculez les valeurs fréquemment utilisées
- ✅ Créez des index sur tous les champs de requête
- ✅ Utilisez des transactions pour les opérations critiques
- ✅ La flexibilité doit être contrôlée, pas anarchique
- ✅ Gérez la duplication avec une stratégie claire

---


⏭️ [Limite de taille des documents (16 Mo)](/04-modelisation-des-donnees/08-limite-taille-documents.md)
