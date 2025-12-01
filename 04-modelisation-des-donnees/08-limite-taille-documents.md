🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.8 Limite de taille des documents (16 Mo)

## Introduction

L'une des contraintes fondamentales de MongoDB est la **limite de 16 Mo par document**. Cette limite, loin d'être arbitraire, est une décision d'architecture qui influence profondément la façon dont vous devez modéliser vos données.

Comprendre cette limite, savoir pourquoi elle existe, comment la mesurer et comment concevoir vos schémas pour l'éviter est essentiel pour construire des applications MongoDB robustes et performantes.

Dans ce chapitre, nous allons explorer en détail cette contrainte, ses implications pratiques et les stratégies pour la gérer efficacement.

---

## La limite de 16 Mo : qu'est-ce que c'est ?

### Définition précise

MongoDB impose une **taille maximale de 16 777 216 octets (16 Mo)** pour un document BSON unique. Cette limite s'applique à :

- **Le document complet** incluant tous ses champs et sous-documents
- **Les tableaux imbriqués** avec tous leurs éléments
- **Les métadonnées BSON** (noms de champs, types, etc.)

**Important :** Cette limite concerne la représentation **BSON** (Binary JSON) du document, pas la représentation JSON. Le BSON ajoute des métadonnées qui augmentent légèrement la taille.

### Conversion des unités

Pour mieux comprendre :

```
16 Mo = 16 777 216 octets
     = 16 384 Ko
     = 16 Mo
```

**En pratique :**
- Environ **16 millions de caractères** en texte brut
- Environ **2 000 à 4 000 pages** de texte
- Environ **500 à 1 000 documents Word** moyens
- Environ **50 000 à 100 000 petits objets JSON**

---

## Pourquoi cette limite existe-t-elle ?

### 1. Performance de la mémoire

**Raison principale :** MongoDB charge les documents complets en mémoire RAM pour les traiter.

```
Document de 16 Mo × 1000 requêtes simultanées = 16 Go de RAM
```

Si les documents étaient illimités, quelques requêtes simultanées pourraient **saturer toute la mémoire** du serveur.

### 2. Performance du réseau

**Transfert de données :**

```
Document de 100 Mo × réseau 1 Gbps = 0.8 secondes de transfert minimum
Document de 16 Mo × réseau 1 Gbps = 0.13 secondes de transfert
```

Des documents plus grands :
- Augmentent la **latence réseau**
- Consomment plus de **bande passante**
- Ralentissent les **opérations distribuées** (réplication, sharding)

### 3. Atomicité et performances d'écriture

MongoDB garantit l'**atomicité au niveau du document**. Chaque modification de document doit :

1. Être écrite dans le journal (write-ahead log)
2. Être répliquée vers les secondaires
3. Potentiellement réorganiser le document sur le disque

Plus le document est grand :
- Plus ces opérations sont **coûteuses**
- Plus le risque de **fragmentation** est élevé
- Plus la **journalisation** est volumineuse

### 4. Design encouragé

Cette limite **encourage les bonnes pratiques** :
- Normalisation quand nécessaire
- Utilisation de références pour les relations one-to-many volumineuses
- Séparation des préoccupations
- Éviter l'accumulation infinie de données

---

## Comment mesurer la taille d'un document ?

### Méthode 1 : Object.bsonsize() dans mongosh

```javascript
// Vérifier la taille d'un document
const doc = db.articles.findOne({ _id: ObjectId("...") })
const tailleBSON = Object.bsonsize(doc)

print(`Taille du document : ${tailleBSON} octets`)
print(`Taille en Ko : ${(tailleBSON / 1024).toFixed(2)} Ko`)
print(`Taille en Mo : ${(tailleBSON / 1024 / 1024).toFixed(2)} Mo`)

// Calculer le pourcentage de la limite
const pourcentage = (tailleBSON / 16777216 * 100).toFixed(2)
print(`Utilisation : ${pourcentage}% de la limite`)
```

**Exemple de sortie :**
```
Taille du document : 2457600 octets
Taille en Ko : 2400.00 Ko
Taille en Mo : 2.34 Mo
Utilisation : 14.65% de la limite
```

### Méthode 2 : Trouver les plus gros documents

```javascript
// Trouver les 10 documents les plus volumineux
db.articles.find().forEach(doc => {
  const size = Object.bsonsize(doc)
  if (size > 1024 * 1024) {  // Plus de 1 Mo
    print(`Document ${doc._id} : ${(size / 1024 / 1024).toFixed(2)} Mo`)
  }
})
```

### Méthode 3 : Agrégation pour analyser

```javascript
// Statistiques sur la taille des documents
db.articles.aggregate([
  {
    $project: {
      taille: { $bsonSize: "$$ROOT" }  // Taille du document complet
    }
  },
  {
    $group: {
      _id: null,
      tailleMoyenne: { $avg: "$taille" },
      tailleMax: { $max: "$taille" },
      tailleMin: { $min: "$taille" }
    }
  }
])
```

**Résultat exemple :**
```javascript
{
  _id: null,
  tailleMoyenne: 125430,      // ~122 Ko en moyenne
  tailleMax: 8945231,         // ~8.5 Mo maximum
  tailleMin: 512              // 512 octets minimum
}
```

### Méthode 4 : Analyser un champ spécifique

```javascript
// Taille d'un tableau spécifique
db.articles.aggregate([
  {
    $project: {
      titre: 1,
      nombreCommentaires: { $size: "$commentaires" },
      tailleCommentaires: { $bsonSize: "$commentaires" }
    }
  },
  { $sort: { tailleCommentaires: -1 } },
  { $limit: 10 }
])
```

---

## Quand la limite devient-elle un problème ?

### Scénarios à risque

#### 1. Collections avec tableaux croissants

**Problème typique :** Tableaux qui accumulent sans limite.

```javascript
// ⚠️ DANGER : Tableau de commentaires qui grandit indéfiniment
{
  _id: ObjectId("..."),
  titre: "Article viral",
  contenu: "...",
  commentaires: [
    // Commence avec 10 commentaires (5 Ko)
    // Après 1 an : 5 000 commentaires (2.5 Mo)
    // Après 5 ans : 25 000 commentaires (12.5 Mo) ← Proche de la limite !
    // Après 7 ans : 35 000 commentaires → ❌ ERREUR 16 Mo !
  ]
}
```

**Signes d'alerte :**
- Tableaux dans `posts`, `commentaires`, `likes`, `vues`, `historique`
- Collections d'événements temporels sans limite
- Logs ou métriques imbriqués

#### 2. Données binaires volumineuses

**Problème :** Stocker directement des fichiers dans les documents.

```javascript
// ⚠️ DANGER : Image encodée en base64
{
  _id: ObjectId("..."),
  titre: "Photo de profil",
  imageBase64: "iVBORw0KGgoAAAANSUhEUgAA..."  // ← 5 Mo en base64 !
}

// ⚠️ DANGER : Document PDF
{
  _id: ObjectId("..."),
  titre: "Contrat",
  pdfData: BinData(0, "JVBERi0xLjQKJ...")  // ← 20 Mo → IMPOSSIBLE !
}
```

**Règle :** Ne **jamais** stocker de fichiers volumineux directement dans les documents.

#### 3. Historiques complets

**Problème :** Conserver tout l'historique dans le document.

```javascript
// ⚠️ DANGER : Historique de modifications illimité
{
  _id: ObjectId("..."),
  titre: "Document collaboratif",
  contenu: "Contenu actuel...",
  historique: [
    { date: ISODate("2020-01-01"), auteur: "user1", contenu: "Version 1..." },
    { date: ISODate("2020-01-02"), auteur: "user2", contenu: "Version 2..." },
    // ... 10 000 révisions plus tard → 14 Mo !
  ]
}
```

#### 4. Imbrication profonde

**Problème :** Structures très imbriquées avec beaucoup de données.

```javascript
// ⚠️ DANGER : Catalogue produit avec toutes les variantes
{
  _id: ObjectId("..."),
  nom: "T-shirt",
  variantes: [
    {
      taille: "S",
      couleurs: [
        {
          nom: "Rouge",
          images: [ /* 50 images haute résolution */ ],
          stock: { /* détails par entrepôt */ }
        }
        // × 20 couleurs × 10 tailles = 200 variantes détaillées
      ]
    }
  ]
}
```

---

## Solutions pour gérer les documents volumineux

### Solution 1 : Utiliser des références (Child-Referencing)

**Au lieu de tout imbriquer, séparer en collections.**

#### Avant (anti-pattern) :

```javascript
{
  _id: ObjectId("..."),
  titre: "Article",
  commentaires: [ /* 50 000 commentaires → 15 Mo */ ]
}
```

#### Après (solution) :

```javascript
// Collection "articles"
{
  _id: ObjectId("..."),
  titre: "Article",
  nombreCommentaires: 50000,  // Compteur pour affichage
  datePublication: ISODate("2024-01-15")
}

// Collection "commentaires" (séparée)
{
  _id: ObjectId("..."),
  articleId: ObjectId("..."),
  auteur: "Sophie",
  texte: "Excellent article !",
  date: ISODate("2024-01-16")
}

// Index pour performance
db.commentaires.createIndex({ articleId: 1, date: -1 })
```

**Avantages :**
- ✅ Pas de limite sur le nombre de commentaires
- ✅ Pagination facile
- ✅ Performance maintenue

---

### Solution 2 : Pattern Subset (Top N)

**Imbriquer seulement les N éléments les plus importants.**

```javascript
{
  _id: ObjectId("..."),
  titre: "Produit",
  avisRecents: [  // ← Seulement les 10 derniers avis
    { auteur: "Jean", note: 5, texte: "Excellent !" },
    { auteur: "Marie", note: 4, texte: "Très bien" }
    // ... 8 autres avis
  ],
  nombreAvisTotal: 5234,
  statistiques: {
    noteMoyenne: 4.3,
    distribution: { 5: 3200, 4: 1500, 3: 400, 2: 100, 1: 34 }
  }
}

// Collection "avis" (tous les avis)
{
  _id: ObjectId("..."),
  produitId: ObjectId("..."),
  auteur: "Jean",
  note: 5,
  texte: "Excellent produit !",
  date: ISODate("2024-01-15")
}
```

**Avantages :**
- ✅ Affichage rapide avec les avis les plus pertinents
- ✅ Document principal reste petit
- ✅ Tous les avis accessibles via la collection séparée

---

### Solution 3 : Pattern Bucket (Regroupement)

**Regrouper les éléments en "seaux" de taille fixe.**

#### Avant (anti-pattern) :

```javascript
// Un document par mesure IoT → 1 million de documents/heure !
{
  _id: ObjectId("..."),
  capteurId: "SENSOR-001",
  temperature: 22.5,
  timestamp: ISODate("2024-01-15T10:00:00Z")
}
```

#### Après (solution) :

```javascript
// Regroupement par heure (bucket)
{
  _id: ObjectId("..."),
  capteurId: "SENSOR-001",
  date: ISODate("2024-01-15T10:00:00Z"),
  mesures: [
    { timestamp: ISODate("2024-01-15T10:00:00Z"), temperature: 22.5 },
    { timestamp: ISODate("2024-01-15T10:01:00Z"), temperature: 22.6 },
    { timestamp: ISODate("2024-01-15T10:02:00Z"), temperature: 22.4 }
    // ... 60 mesures max (une par minute)
  ],
  statistiques: {
    nombreMesures: 60,
    temperatureMoyenne: 22.5,
    temperatureMin: 22.1,
    temperatureMax: 22.9
  }
}
```

**Avantages :**
- ✅ 60 mesures par document au lieu de 60 documents
- ✅ Réduction de 98% du nombre de documents
- ✅ Meilleure performance d'indexation
- ✅ Document ne dépassera jamais ~50 Ko

---

### Solution 4 : GridFS pour fichiers volumineux

**GridFS** divise les fichiers en chunks de 255 Ko.

#### Utilisation de GridFS :

```javascript
// Stocker un fichier avec GridFS
const bucket = new GridFSBucket(db)

// Upload
const uploadStream = bucket.openUploadStream('mon-fichier.pdf', {
  metadata: {
    type: 'document',
    utilisateur: 'sophie@example.com',
    dateUpload: new Date()
  }
})

fs.createReadStream('fichier.pdf').pipe(uploadStream)

uploadStream.on('finish', () => {
  console.log(`Fichier uploadé : ${uploadStream.id}`)
})

// Document principal référence le fichier GridFS
{
  _id: ObjectId("..."),
  titre: "Contrat de service",
  description: "Contrat signé avec le client ABC",
  fichierGridFSId: uploadStream.id,  // ← Référence
  tailleFichier: 5242880,  // 5 Mo
  dateCreation: ISODate("2024-01-15")
}

// Download
const downloadStream = bucket.openDownloadStream(fileId)
downloadStream.pipe(fs.createWriteStream('fichier-telecharge.pdf'))
```

**Quand utiliser GridFS :**
- Fichiers > 16 Mo (obligatoire)
- Fichiers entre 1 Mo et 16 Mo (recommandé)
- Images, PDFs, vidéos, archives

**Avantages :**
- ✅ Pas de limite de taille (fichiers de plusieurs Go possibles)
- ✅ Streaming efficace
- ✅ Métadonnées associées au fichier
- ✅ Réplication automatique avec MongoDB

---

### Solution 5 : Compression des données

**Compresser les champs texte volumineux.**

```javascript
// Avec compression (exemple conceptuel - nécessite une bibliothèque)
const contenuCompresse = compresser(contenuLong)

{
  _id: ObjectId("..."),
  titre: "Article",
  contenuCompresse: BinData(0, contenuCompresse),  // ← Compressé
  tailleOriginale: 5242880,
  tailleCompresse: 1048576,
  compression: "gzip"
}

// Décompression lors de la lecture
const contenu = decompresser(document.contenuCompresse)
```

**Cas d'usage :**
- Logs textuels volumineux
- Contenu riche (HTML, Markdown)
- Données JSON imbriquées

**Important :** La compression ajoute de la complexité. À utiliser seulement si nécessaire.

---

### Solution 6 : Archivage et Time-To-Live (TTL)

**Supprimer automatiquement les vieilles données.**

```javascript
// Index TTL : suppression automatique après 30 jours
db.logs.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 2592000 }  // 30 jours
)

// Document se supprime automatiquement
{
  _id: ObjectId("..."),
  message: "Log entry",
  niveau: "INFO",
  createdAt: ISODate("2024-01-15T10:00:00Z")
  // ← Supprimé automatiquement le 2024-02-14
}
```

**Ou archivage manuel vers collection historique :**

```javascript
// Archiver les données > 1 an dans collection séparée
const unAnAuparavant = new Date()
unAnAuparavant.setFullYear(unAnAuparavant.getFullYear() - 1)

// Copier vers archive
const anciensDocuments = db.articles.find({
  datePublication: { $lt: unAnAuparavant }
})

anciensDocuments.forEach(doc => {
  db.articlesArchives.insertOne(doc)
})

// Supprimer de la collection principale
db.articles.deleteMany({
  datePublication: { $lt: unAnAuparavant }
})
```

---

### Solution 7 : Stockage externe (S3, Cloud Storage)

**Stocker les fichiers dans un service de stockage externe.**

```javascript
// Document MongoDB (métadonnées seulement)
{
  _id: ObjectId("..."),
  titre: "Vidéo de présentation",
  description: "Présentation produit 2024",
  urlS3: "https://mon-bucket.s3.amazonaws.com/videos/presentation-2024.mp4",
  tailleFichier: 524288000,  // 500 Mo
  duree: 600,  // 10 minutes
  format: "mp4",
  dateUpload: ISODate("2024-01-15")
}
```

**Avantages :**
- ✅ Pas de limite de taille
- ✅ CDN pour diffusion rapide
- ✅ Coût optimisé pour le stockage
- ✅ Spécialisé pour les fichiers statiques

**Services courants :**
- AWS S3
- Google Cloud Storage
- Azure Blob Storage
- Cloudinary (images/vidéos)

---

## Stratégies d'optimisation

### 1. Analyser et identifier les gros documents

**Script de monitoring :**

```javascript
// Fonction pour trouver les documents volumineux
function trouverDocumentsVolumineux(collection, seuilMo) {
  const seuilOctets = seuilMo * 1024 * 1024
  const resultats = []

  db[collection].find().forEach(doc => {
    const taille = Object.bsonsize(doc)
    if (taille > seuilOctets) {
      resultats.push({
        _id: doc._id,
        taille: taille,
        tailleMo: (taille / 1024 / 1024).toFixed(2),
        pourcentageLimit: ((taille / 16777216) * 100).toFixed(2)
      })
    }
  })

  return resultats.sort((a, b) => b.taille - a.taille)
}

// Utilisation
const grosDocuments = trouverDocumentsVolumineux("articles", 1)
printjson(grosDocuments)
```

### 2. Identifier les champs volumineux

```javascript
// Analyser quel champ prend le plus de place
db.articles.aggregate([
  { $limit: 100 },  // Échantillon
  {
    $project: {
      tailleCommentaires: { $bsonSize: "$commentaires" },
      tailleContenu: { $bsonSize: "$contenu" },
      tailleMetadonnees: { $bsonSize: "$metadonnees" },
      tailleTotal: { $bsonSize: "$$ROOT" }
    }
  }
])
```

### 3. Établir des alertes

```javascript
// Script de monitoring quotidien
function verifierTailleDocuments() {
  const seuil = 10 * 1024 * 1024  // 10 Mo

  db.articles.find().forEach(doc => {
    const taille = Object.bsonsize(doc)

    if (taille > seuil) {
      // Logger ou envoyer une alerte
      print(`⚠️  ALERTE : Document ${doc._id} = ${(taille / 1024 / 1024).toFixed(2)} Mo`)

      // Potentiellement déclencher une action automatique
      // (archivage, notification, etc.)
    }
  })
}

// Exécuter quotidiennement
verifierTailleDocuments()
```

---

## Erreurs liées à la limite de 16 Mo

### Message d'erreur typique

```javascript
MongoServerError: BSONObj size: 17825792 (0x10FE000) is invalid.
Size must be between 0 and 16793600(16MB)
First element: _id: ObjectId('...')
```

### Que faire quand vous rencontrez cette erreur ?

#### 1. Identifier le document problématique

```javascript
// L'erreur vous donne l'_id, récupérez le document
const doc = db.articles.findOne({ _id: ObjectId("...") })

// Vérifier sa taille
const taille = Object.bsonsize(doc)
print(`Taille : ${(taille / 1024 / 1024).toFixed(2)} Mo`)

// Identifier les gros champs
print(`Commentaires : ${Object.bsonsize(doc.commentaires)} octets`)
print(`Contenu : ${Object.bsonsize(doc.contenu)} octets`)
```

#### 2. Solutions d'urgence

**Option A : Extraire les données vers une collection séparée**

```javascript
// Sauvegarder les commentaires ailleurs
doc.commentaires.forEach(commentaire => {
  db.commentairesArticle.insertOne({
    articleId: doc._id,
    ...commentaire
  })
})

// Vider le tableau dans le document original
db.articles.updateOne(
  { _id: doc._id },
  {
    $set: { commentaires: [] },
    $inc: { version: 1 }
  }
)
```

**Option B : Archiver une partie des données**

```javascript
// Garder seulement les 100 derniers commentaires
const commentairesRecents = doc.commentaires.slice(-100)
const commentairesArchives = doc.commentaires.slice(0, -100)

// Archiver les anciens
db.commentairesArchives.insertOne({
  articleId: doc._id,
  commentaires: commentairesArchives,
  dateArchivage: new Date()
})

// Mettre à jour le document
db.articles.updateOne(
  { _id: doc._id },
  { $set: { commentaires: commentairesRecents } }
)
```

---

## Bonnes pratiques

### ✅ À faire

1. **Anticiper la croissance**
   - Calculer la taille maximale théorique d'un document
   - Prévoir l'évolution sur 1 an, 5 ans, 10 ans

2. **Monitorer régulièrement**
   - Script quotidien pour identifier les documents > 5 Mo
   - Alertes si documents > 10 Mo

3. **Limiter les tableaux**
   - Imposer une limite applicative (ex : max 100 éléments)
   - Utiliser des références au-delà de cette limite

4. **Documenter les choix**
   - Expliquer pourquoi tel champ est imbriqué
   - Documenter les stratégies de croissance

5. **Tester avec des données réalistes**
   - Ne pas tester qu'avec 10 documents
   - Simuler 1 an, 5 ans de données

### ⚠️ À éviter

1. **Assumer que "ça ira"**
   - Un tableau de 10 éléments aujourd'hui → 10 000 dans 5 ans

2. **Ignorer les alertes**
   - Document à 8 Mo → sera à 16 Mo bientôt

3. **Stocker des fichiers directement**
   - Toujours utiliser GridFS ou stockage externe

4. **Imbrication sans limite**
   - Tout imbriquer "parce que c'est MongoDB"

5. **Ne pas mesurer**
   - "Je pense que c'est petit" → Mesurer !

---

## Tableau récapitulatif des solutions

| Problème | Solution recommandée | Complexité | Performance |
|----------|---------------------|------------|-------------|
| Commentaires illimités | Child-Referencing | Faible | Excellente |
| Top N + total | Pattern Subset | Faible | Excellente |
| Données temporelles | Pattern Bucket | Moyenne | Très bonne |
| Fichiers > 16 Mo | GridFS | Moyenne | Bonne |
| Fichiers 1-16 Mo | GridFS ou S3 | Faible-Moyenne | Excellente |
| Historique complet | Collection séparée | Faible | Bonne |
| Logs volumineux | TTL + Archivage | Faible | Bonne |
| Images/Vidéos | Stockage externe (S3) | Faible | Excellente |
| Contenu texte long | Compression | Élevée | Moyenne |

---

## Outils de diagnostic

### MongoDB Compass

MongoDB Compass affiche la taille des documents visuellement :

```
Documents > Schema > Analyze
→ Document size distribution
→ Field size analysis
```

### Scripts de monitoring

```javascript
// Script complet de diagnostic
function diagnosticTailleDocuments(nomCollection) {
  print(`\n=== Diagnostic : ${nomCollection} ===\n`)

  const stats = db[nomCollection].stats()
  print(`Nombre de documents : ${stats.count}`)
  print(`Taille moyenne : ${(stats.avgObjSize / 1024).toFixed(2)} Ko`)
  print(`Taille totale : ${(stats.size / 1024 / 1024).toFixed(2)} Mo\n`)

  // Top 10 des plus gros documents
  print("Top 10 documents les plus volumineux :")

  const gros = []
  db[nomCollection].find().limit(1000).forEach(doc => {
    gros.push({
      _id: doc._id,
      taille: Object.bsonsize(doc)
    })
  })

  gros.sort((a, b) => b.taille - a.taille)
  gros.slice(0, 10).forEach((item, index) => {
    const tailleMo = (item.taille / 1024 / 1024).toFixed(2)
    const pourcentage = ((item.taille / 16777216) * 100).toFixed(2)
    print(`${index + 1}. ${item._id} : ${tailleMo} Mo (${pourcentage}%)`)
  })
}

// Utilisation
diagnosticTailleDocuments("articles")
```

---

## Conclusion

La **limite de 16 Mo** est une contrainte fondamentale de MongoDB qui :

- ✅ **Encourage** les bonnes pratiques de modélisation
- ✅ **Protège** les performances et la stabilité du système
- ✅ **Force** à penser scalabilité dès la conception

**Règles d'or :**

1. 📏 **Mesurez** : Utilisez `Object.bsonsize()` régulièrement
2. 📏 **Anticipez** : Calculez la taille maximale théorique
3. 📏 **Limitez** : Imposez des limites applicatives sur les tableaux
4. 📏 **Séparez** : Utilisez des références pour les données volumineuses
5. 📏 **Surveillez** : Établissez des alertes à 10 Mo
6. 📏 **GridFS** : Pour tous les fichiers
7. 📏 **Documentez** : Expliquez vos choix de conception

En respectant ces principes, la limite de 16 Mo ne sera jamais un obstacle, mais un guide pour concevoir des schémas MongoDB efficaces et performants.

---

**Points clés à retenir :**

- ✅ Limite stricte de 16 Mo (16 777 216 octets) par document
- ✅ Cette limite existe pour protéger les performances
- ✅ Utilisez Object.bsonsize() pour mesurer la taille
- ✅ Les tableaux illimités sont le piège le plus courant
- ✅ Pattern Subset = solution élégante pour top N + total
- ✅ GridFS = solution pour fichiers > 16 Mo
- ✅ Stockage externe (S3) = idéal pour médias
- ✅ Établissez des alertes à 10 Mo pour anticiper

---


⏭️ [Conception pour la performance](/04-modelisation-des-donnees/09-conception-performance.md)
