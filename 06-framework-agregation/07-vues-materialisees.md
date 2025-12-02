🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7 Vues (Views) et Vues Matérialisées

## Introduction

Les **vues** (views) dans MongoDB sont comme des "fenêtres" ou des "raccourcis" vers vos données. Elles vous permettent de créer des représentations virtuelles de vos collections, transformées selon vos besoins, sans dupliquer les données.

### Analogie Simple

Imaginez une bibliothèque avec des milliers de livres (votre collection). Plutôt que de créer des copies physiques organisées différemment pour chaque besoin, vous créez des **catalogues** (vues) :
- 📚 Catalogue "Nouveautés 2024" → Vue filtrée par date
- 📚 Catalogue "Meilleures ventes" → Vue triée par popularité
- 📚 Catalogue "Romans en français" → Vue filtrée par langue et genre

Les livres restent au même endroit, mais vous avez différentes façons de les consulter !

## Qu'est-ce qu'une Vue ?

### Définition

Une **vue** dans MongoDB est :
- 🔍 Une collection "virtuelle" basée sur un pipeline d'agrégation
- 📖 Lecture seule (read-only)
- 🔄 Dynamique : reflète toujours les données actuelles
- 💾 Ne stocke PAS de données (juste la définition du pipeline)

### Vue vs Collection Normale

| Caractéristique | Collection Normale | Vue |
|-----------------|-------------------|-----|
| **Stockage** | Stocke les données physiquement | Ne stocke que la définition |
| **Taille** | Occupe de l'espace disque | Occupe très peu d'espace |
| **Modification** | Lecture et écriture | Lecture seule |
| **Données** | Statiques jusqu'à modification | Toujours à jour |
| **Performance** | Accès direct | Exécute le pipeline à chaque requête |

### Comment Ça Marche ?

```
┌─────────────────────────────────────────┐
│   Collection Source: produits           │
│   (1 million de documents)              │
└──────────────┬──────────────────────────┘
               │
               │  Pipeline défini dans la vue:
               │  [
               │    { $match: { actif: true } },
               │    { $sort: { prix: 1 } }
               │  ]
               │
               ↓
┌─────────────────────────────────────────┐
│   Vue: produits_actifs                  │
│   (Vue virtuelle - pas de stockage)     │
│   Exécute le pipeline quand consultée   │
└─────────────────────────────────────────┘
```

## Créer une Vue

### Syntaxe de Base

```javascript
db.createView(
  "nom_de_la_vue",           // Nom de la vue
  "collection_source",       // Collection de base
  [ /* pipeline */ ]         // Pipeline d'agrégation
)
```

### Exemple Simple

**Collection source : produits**
```javascript
{
  "_id": 1,
  "nom": "Ordinateur",
  "prix": 1200,
  "stock": 5,
  "actif": true,
  "categorie": "Électronique"
}
```

**Créer une vue des produits actifs :**
```javascript
db.createView(
  "produits_actifs",          // Nom de la vue
  "produits",                 // Collection source
  [
    { $match: { actif: true } },
    { $sort: { prix: 1 } },
    { $project: {
        nom: 1,
        prix: 1,
        stock: 1,
        categorie: 1
      }
    }
  ]
)
```

### Utiliser une Vue

Une fois créée, la vue s'utilise comme une collection normale (en lecture seule) :

```javascript
// Lire la vue
db.produits_actifs.find()

// Avec des filtres supplémentaires
db.produits_actifs.find({ prix: { $lt: 500 } })

// Compter les documents
db.produits_actifs.countDocuments()

// Avec limit et sort
db.produits_actifs.find().limit(10).sort({ prix: -1 })
```

**Important :** Vous NE pouvez PAS :
```javascript
// ❌ ERREUR - Les vues sont en lecture seule
db.produits_actifs.insertOne({ ... })
db.produits_actifs.updateOne({ ... })
db.produits_actifs.deleteOne({ ... })
```

## Cas d'Usage des Vues

### 1. Simplifier les Requêtes Complexes

**Sans vue :**
```javascript
// À chaque fois, réécrire le pipeline complexe
db.commandes.aggregate([
  { $match: { statut: "payé", date: { $gte: ISODate("2024-01-01") } } },
  { $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },
  { $unwind: "$client" },
  { $project: {
      numero: 1,
      montant: 1,
      date: 1,
      "client.nom": 1,
      "client.email": 1
    }
  }
])
```

**Avec vue :**
```javascript
// Créer la vue une fois
db.createView(
  "commandes_payees_enrichies",
  "commandes",
  [
    { $match: { statut: "payé", date: { $gte: ISODate("2024-01-01") } } },
    { $lookup: {
        from: "clients",
        localField: "clientId",
        foreignField: "_id",
        as: "client"
      }
    },
    { $unwind: "$client" },
    { $project: {
        numero: 1,
        montant: 1,
        date: 1,
        "client.nom": 1,
        "client.email": 1
      }
    }
  ]
)

// Utiliser simplement
db.commandes_payees_enrichies.find()
db.commandes_payees_enrichies.find({ montant: { $gt: 1000 } })
```

### 2. Masquer des Données Sensibles

**Collection originale : utilisateurs**
```javascript
{
  "_id": 1,
  "nom": "Alice Dupont",
  "email": "alice@example.com",
  "motDePasse": "$2a$10$...",      // Sensible
  "numeroSecu": "1 85 12 ...",     // Sensible
  "adresse": "123 rue de Paris",
  "telephone": "+33612345678"
}
```

**Vue publique (sans données sensibles) :**
```javascript
db.createView(
  "utilisateurs_publics",
  "utilisateurs",
  [
    {
      $project: {
        nom: 1,
        email: 1,
        ville: { $arrayElemAt: [{ $split: ["$adresse", ","] }, -1] },
        // Pas de motDePasse, pas de numeroSecu
        _id: 1
      }
    }
  ]
)
```

**Utilisation :**
```javascript
// Applications frontend peuvent utiliser cette vue
db.utilisateurs_publics.find()
// Résultat : { "_id": 1, "nom": "Alice Dupont", "email": "alice@...", "ville": "Paris" }
```

### 3. Créer des Rapports Préconfigurés

**Vue : Rapport mensuel des ventes**
```javascript
db.createView(
  "rapport_ventes_mensuelles",
  "ventes",
  [
    {
      $group: {
        _id: {
          annee: { $year: "$date" },
          mois: { $month: "$date" }
        },
        totalVentes: { $sum: "$montant" },
        nombreTransactions: { $sum: 1 },
        venteMoyenne: { $avg: "$montant" }
      }
    },
    {
      $sort: { "_id.annee": -1, "_id.mois": -1 }
    },
    {
      $project: {
        _id: 0,
        periode: {
          $concat: [
            { $toString: "$_id.annee" },
            "-",
            { $cond: {
                if: { $lt: ["$_id.mois", 10] },
                then: { $concat: ["0", { $toString: "$_id.mois" }] },
                else: { $toString: "$_id.mois" }
              }
            }
          ]
        },
        totalVentes: { $round: ["$totalVentes", 2] },
        nombreTransactions: 1,
        venteMoyenne: { $round: ["$venteMoyenne", 2] }
      }
    }
  ]
)

// Utilisation simple
db.rapport_ventes_mensuelles.find()
```

### 4. Dénormaliser des Données pour l'Application

**Vue : Produits avec catégorie enrichie**
```javascript
db.createView(
  "produits_avec_categorie",
  "produits",
  [
    {
      $lookup: {
        from: "categories",
        localField: "categorieId",
        foreignField: "_id",
        as: "categorie"
      }
    },
    { $unwind: "$categorie" },
    {
      $project: {
        nom: 1,
        prix: 1,
        stock: 1,
        categorieName: "$categorie.nom",
        categorieDescription: "$categorie.description"
      }
    }
  ]
)
```

## Avantages des Vues

### ✅ Avantages

1. **Simplification**
   - Encapsule la complexité
   - Code plus lisible et maintenable
   - Réutilisation facile

2. **Sécurité**
   - Masque les données sensibles
   - Contrôle d'accès granulaire
   - Pas de duplication de données sensibles

3. **Cohérence**
   - Une seule définition de la logique
   - Changements propagés automatiquement
   - Évite les erreurs de duplication

4. **Flexibilité**
   - Modification facile du pipeline
   - Pas d'impact sur les applications existantes
   - Possibilité de créer plusieurs vues de la même collection

5. **Économie d'Espace**
   - Pas de duplication de données
   - Seulement la définition du pipeline est stockée

### ⚠️ Limitations

1. **Lecture Seule**
   - Impossible d'insérer, modifier ou supprimer
   - Seulement pour consulter les données

2. **Performance**
   - Le pipeline est exécuté à chaque requête
   - Pas de cache automatique
   - Peut être lent sur de gros volumes

3. **Pas d'Index Direct**
   - Les vues n'ont pas leurs propres index
   - Utilise les index de la collection source
   - Performance dépend des index source

## Vues Matérialisées

### Qu'est-ce qu'une Vue Matérialisée ?

Contrairement aux vues normales (virtuelles), les **vues matérialisées** :
- 💾 **Stockent réellement les données** sur le disque
- ⚡ **Sont plus rapides** à consulter (pas de calcul à la volée)
- 🔄 **Nécessitent une actualisation** manuelle ou planifiée
- 📊 **Occupent de l'espace disque**

### Analogie

**Vue normale** = Calculatrice
- Calcule à chaque fois qu'on lui demande
- Pas de mémoire des résultats précédents

**Vue matérialisée** = Rapport Excel enregistré
- Résultats déjà calculés et sauvegardés
- Consultation instantanée
- Doit être régénéré pour être à jour

### MongoDB ne Supporte pas Nativement les Vues Matérialisées

MongoDB n'a pas de concept natif de "vues matérialisées" comme PostgreSQL ou Oracle. Cependant, on peut les **simuler** avec `$out` ou `$merge`.

## Créer une Vue Matérialisée avec $out

### Principe

`$out` écrit les résultats du pipeline dans une nouvelle collection.

### Syntaxe

```javascript
db.collection_source.aggregate([
  /* pipeline d'agrégation */,
  { $out: "nom_collection_destination" }
])
```

### Exemple : Rapport de Ventes Matérialisé

```javascript
// Créer/mettre à jour la vue matérialisée
db.ventes.aggregate([
  // Pipeline de transformation
  {
    $group: {
      _id: {
        produit: "$produitId",
        mois: { $month: "$date" },
        annee: { $year: "$date" }
      },
      totalVentes: { $sum: "$montant" },
      quantiteVendue: { $sum: "$quantite" },
      nombreTransactions: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      produitId: "$_id.produit",
      mois: "$_id.mois",
      annee: "$_id.annee",
      totalVentes: 1,
      quantiteVendue: 1,
      nombreTransactions: 1,
      venteMoyenne: { $divide: ["$totalVentes", "$nombreTransactions"] }
    }
  },
  {
    $sort: { annee: -1, mois: -1 }
  },

  // Écrire dans une collection
  { $out: "ventes_mensuelles_par_produit" }
])

// Maintenant, utiliser comme une collection normale
db.ventes_mensuelles_par_produit.find()
db.ventes_mensuelles_par_produit.find({ annee: 2024 })

// Créer des index sur la vue matérialisée
db.ventes_mensuelles_par_produit.createIndex({ annee: -1, mois: -1 })
db.ventes_mensuelles_par_produit.createIndex({ produitId: 1 })
```

### Comportement de $out

**Important :**
- ⚠️ `$out` **REMPLACE** complètement la collection de destination
- 🗑️ Toutes les données précédentes sont perdues
- ✍️ Crée la collection si elle n'existe pas

```javascript
// Première exécution
db.ventes.aggregate([
  { $match: { ... } },
  { $out: "rapport" }
])
// → Crée "rapport" avec les résultats

// Deuxième exécution
db.ventes.aggregate([
  { $match: { ... } },
  { $out: "rapport" }
])
// → REMPLACE complètement "rapport" avec les nouveaux résultats
```

## Créer une Vue Matérialisée avec $merge

### Avantages de $merge

`$merge` est plus flexible que `$out` :
- ✅ **Met à jour** les documents existants au lieu de tout remplacer
- ✅ **Ajoute** de nouveaux documents
- ✅ **Personnalisable** : contrôle sur le comportement de fusion

### Syntaxe de Base

```javascript
db.collection_source.aggregate([
  /* pipeline */,
  {
    $merge: {
      into: "collection_destination",
      on: "_id",  // Champ(s) pour identifier les documents
      whenMatched: "merge",  // Ou "replace", "keepExisting", "fail"
      whenNotMatched: "insert"  // Ou "discard", "fail"
    }
  }
])
```

### Options de $merge

| Option | Valeurs | Description |
|--------|---------|-------------|
| **into** | String | Nom de la collection destination |
| **on** | String ou Array | Champ(s) utilisés pour matcher les documents |
| **whenMatched** | merge, replace, keepExisting, fail, pipeline | Action si le document existe |
| **whenNotMatched** | insert, discard, fail | Action si le document n'existe pas |

### Exemple : Mise à Jour Incrémentale

**Scénario :** Actualiser quotidiennement les statistiques sans perdre les données historiques.

```javascript
// Exécuté chaque jour
db.ventes.aggregate([
  // Ventes d'aujourd'hui
  {
    $match: {
      date: {
        $gte: new Date(new Date().setHours(0, 0, 0, 0)),
        $lt: new Date(new Date().setHours(23, 59, 59, 999))
      }
    }
  },

  // Agréger par produit
  {
    $group: {
      _id: "$produitId",
      totalVentesAujourdhui: { $sum: "$montant" },
      quantiteVendueAujourdhui: { $sum: "$quantite" }
    }
  },

  // Fusionner avec la collection existante
  {
    $merge: {
      into: "statistiques_produits",
      on: "_id",
      whenMatched: [
        // Pipeline de mise à jour
        {
          $set: {
            totalVentes: { $add: ["$totalVentes", "$$new.totalVentesAujourdhui"] },
            quantiteVendue: { $add: ["$quantiteVendue", "$$new.quantiteVendueAujourdhui"] },
            derniereMAJ: new Date()
          }
        }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

### $merge vs $out

| Caractéristique | $out | $merge |
|-----------------|------|--------|
| **Remplacement** | Remplace tout | Peut mettre à jour |
| **Flexibilité** | Simple, tout ou rien | Très flexible |
| **Incrémental** | Non | Oui |
| **Complexité** | Simple | Plus complexe |
| **Use case** | Rapports complets | Mises à jour incrémentales |

## Actualisation des Vues Matérialisées

Les vues matérialisées doivent être **actualisées régulièrement** pour refléter les nouvelles données.

### Stratégies d'Actualisation

#### 1. Actualisation Manuelle

```javascript
// Fonction pour actualiser la vue
function actualiserRapportVentes() {
  db.ventes.aggregate([
    { $match: { ... } },
    { $group: { ... } },
    { $out: "rapport_ventes" }
  ])
  print("Rapport actualisé à " + new Date())
}

// Appeler manuellement
actualiserRapportVentes()
```

#### 2. Actualisation Planifiée (Cron Job)

**Script Shell (refresh_views.sh) :**
```bash
#!/bin/bash
# Script exécuté quotidiennement par cron

mongosh "mongodb://localhost:27017/mabase" <<EOF
db.ventes.aggregate([
  { \$match: { ... } },
  { \$group: { ... } },
  { \$out: "rapport_ventes" }
])
print("Rapport actualisé")
EOF
```

**Cron (Linux/Mac) :**
```bash
# Exécuter tous les jours à 2h du matin
0 2 * * * /path/to/refresh_views.sh
```

**Task Scheduler (Windows) ou services similaires**

#### 3. Actualisation par Trigger (Change Streams)

**Avec Node.js :**
```javascript
const { MongoClient } = require('mongodb')

async function watchAndRefresh() {
  const client = await MongoClient.connect('mongodb://localhost:27017')
  const db = client.db('mabase')

  // Surveiller les changements sur la collection ventes
  const changeStream = db.collection('ventes').watch()

  let lastRefresh = Date.now()
  const refreshInterval = 5 * 60 * 1000  // 5 minutes

  changeStream.on('change', async (change) => {
    const now = Date.now()

    // Actualiser si assez de temps s'est écoulé
    if (now - lastRefresh > refreshInterval) {
      console.log('Actualisation du rapport...')

      await db.collection('ventes').aggregate([
        { $match: { ... } },
        { $group: { ... } },
        { $out: "rapport_ventes" }
      ]).toArray()

      lastRefresh = now
      console.log('Rapport actualisé')
    }
  })
}

watchAndRefresh()
```

#### 4. Actualisation Incrémentale avec $merge

```javascript
// Actualise seulement les nouvelles données
function actualiserIncrementalement() {
  const dernierActualisation = db.metadata.findOne({
    type: "derniere_actualisation"
  }).date

  db.ventes.aggregate([
    // Seulement les ventes depuis la dernière actualisation
    {
      $match: {
        date: { $gte: dernierActualisation }
      }
    },
    {
      $group: {
        _id: "$produitId",
        nouvellesVentes: { $sum: "$montant" }
      }
    },
    {
      $merge: {
        into: "statistiques_produits",
        on: "_id",
        whenMatched: [
          {
            $set: {
              totalVentes: { $add: ["$totalVentes", "$$new.nouvellesVentes"] }
            }
          }
        ],
        whenNotMatched: "insert"
      }
    }
  ])

  // Mettre à jour la date de dernière actualisation
  db.metadata.updateOne(
    { type: "derniere_actualisation" },
    { $set: { date: new Date() } },
    { upsert: true }
  )
}
```

## Gestion des Vues

### Lister Toutes les Vues

```javascript
// Voir toutes les collections (y compris les vues)
db.getCollectionNames()

// Voir uniquement les vues
db.system.views.find()

// Informations détaillées sur une vue
db.getCollectionInfos({ name: "nom_de_la_vue" })
```

### Modifier une Vue

Pour modifier une vue, il faut la supprimer puis la recréer :

```javascript
// 1. Supprimer la vue
db.produits_actifs.drop()

// 2. Recréer avec la nouvelle définition
db.createView(
  "produits_actifs",
  "produits",
  [
    { $match: { actif: true, stock: { $gt: 0 } } },  // Filtre ajouté
    { $sort: { prix: 1 } }
  ]
)
```

**Ou utiliser collMod (MongoDB 4.2+) :**
```javascript
db.runCommand({
  collMod: "produits_actifs",
  viewOn: "produits",
  pipeline: [
    { $match: { actif: true, stock: { $gt: 0 } } },
    { $sort: { prix: 1 } }
  ]
})
```

### Supprimer une Vue

```javascript
// Supprimer une vue (ou une collection matérialisée)
db.nom_de_la_vue.drop()
```

### Voir la Définition d'une Vue

```javascript
// Obtenir la définition complète
db.getCollectionInfos({ name: "produits_actifs" })[0]

// Résultat :
{
  "name": "produits_actifs",
  "type": "view",
  "options": {
    "viewOn": "produits",
    "pipeline": [
      { "$match": { "actif": true } },
      { "$sort": { "prix": 1 } }
    ]
  }
}
```

## Cas d'Usage Avancés

### 1. Dashboard de Reporting

**Vue matérialisée actualisée chaque nuit :**

```javascript
// Vue : KPIs quotidiens
db.ventes.aggregate([
  {
    $match: {
      date: { $gte: ISODate("2024-01-01") }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d", date: "$date" }
      },
      nombreVentes: { $sum: 1 },
      chiffreAffaires: { $sum: "$montant" },
      panierMoyen: { $avg: "$montant" },
      clientsUniques: { $addToSet: "$clientId" }
    }
  },
  {
    $addFields: {
      nombreClientsUniques: { $size: "$clientsUniques" }
    }
  },
  {
    $project: {
      clientsUniques: 0  // Supprimer le tableau, garder juste le compte
    }
  },
  {
    $sort: { _id: -1 }
  },
  { $out: "kpis_quotidiens" }
])

// Créer des index pour les requêtes rapides
db.kpis_quotidiens.createIndex({ _id: -1 })

// Utilisation ultra-rapide dans le dashboard
db.kpis_quotidiens.find().limit(30)  // Derniers 30 jours
```

### 2. API Publique avec Données Filtrées

**Vue pour API externe :**

```javascript
// Vue : Produits pour l'API publique (sans prix d'achat)
db.createView(
  "produits_api_publique",
  "produits",
  [
    {
      $match: {
        actif: true,
        visible: true
      }
    },
    {
      $lookup: {
        from: "categories",
        localField: "categorieId",
        foreignField: "_id",
        as: "categorie"
      }
    },
    { $unwind: "$categorie" },
    {
      $project: {
        nom: 1,
        description: 1,
        prixVente: 1,  // Seulement le prix de vente
        images: 1,
        stock: { $cond: [{ $gt: ["$stock", 0] }, "Disponible", "Rupture"] },
        categorieName: "$categorie.nom",
        // Pas de prixAchat, pas de fournisseurId
        _id: 1
      }
    }
  ]
)

// L'API utilise cette vue
app.get('/api/produits', async (req, res) => {
  const produits = await db.produits_api_publique.find().toArray()
  res.json(produits)
})
```

### 3. Datawarehouse / Analytics

**Vue matérialisée pour l'analyse :**

```javascript
// Agrégation complexe pour analytics
db.commandes.aggregate([
  { $unwind: "$articles" },
  {
    $lookup: {
      from: "produits",
      localField: "articles.produitId",
      foreignField: "_id",
      as: "produit"
    }
  },
  { $unwind: "$produit" },
  {
    $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },
  { $unwind: "$client" },
  {
    $group: {
      _id: {
        categorie: "$produit.categorie",
        region: "$client.region",
        mois: { $month: "$date" },
        annee: { $year: "$date" }
      },
      nombreVentes: { $sum: 1 },
      chiffreAffaires: { $sum: "$articles.montant" },
      quantiteVendue: { $sum: "$articles.quantite" }
    }
  },
  {
    $project: {
      _id: 0,
      categorie: "$_id.categorie",
      region: "$_id.region",
      mois: "$_id.mois",
      annee: "$_id.annee",
      nombreVentes: 1,
      chiffreAffaires: 1,
      quantiteVendue: 1
    }
  },
  { $merge: {
      into: "analytics_ventes",
      on: ["categorie", "region", "mois", "annee"],
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])

// Index pour requêtes analytics rapides
db.analytics_ventes.createIndex({ annee: -1, mois: -1 })
db.analytics_ventes.createIndex({ categorie: 1, annee: -1 })
db.analytics_ventes.createIndex({ region: 1, annee: -1 })
```

## Bonnes Pratiques

### ✅ Pour les Vues Normales

1. **Utilisez pour simplifier**
   - Encapsulez les pipelines complexes réutilisés
   - Créez des abstractions métier

2. **Attention aux performances**
   - Le pipeline s'exécute à chaque requête
   - Optimisez le pipeline (cf. section 6.6)
   - Utilisez les index de la collection source

3. **Nommage clair**
   ```javascript
   // ✅ BON
   "commandes_payees_2024"
   "produits_actifs_avec_stock"
   "clients_premium_region_paris"

   // ❌ MAUVAIS
   "vue1"
   "temp"
   "data"
   ```

4. **Documentation**
   ```javascript
   // Documenter le but de la vue
   db.createView(
     "rapport_ventes_mensuelles",  // Nom descriptif
     "ventes",
     [
       // Pipeline commenté
       { $match: { statut: "payé" } },  // Seulement ventes validées
       // ...
     ]
   )
   ```

### ✅ Pour les Vues Matérialisées

1. **Planifiez l'actualisation**
   - Définissez une fréquence appropriée
   - Utilisez des cron jobs ou schedulers
   - Considérez l'actualisation incrémentale avec $merge

2. **Créez des index**
   ```javascript
   // Les vues matérialisées sont des collections normales
   db.vue_materialisee.createIndex({ champ_frequemment_recherche: 1 })
   ```

3. **Surveillez l'espace disque**
   ```javascript
   // Vérifier la taille
   db.vue_materialisee.stats()
   ```

4. **Horodatage**
   ```javascript
   // Ajoutez toujours un timestamp de création
   db.ventes.aggregate([
     { $match: { ... } },
     { $group: { ... } },
     {
       $addFields: {
         dateCreation: new Date(),
         version: "1.0"
       }
    },
     { $out: "rapport_ventes" }
   ])
   ```

5. **Gestion des erreurs**
   ```javascript
   try {
     db.ventes.aggregate([
       { $match: { ... } },
       { $out: "rapport_ventes" }
     ])
     console.log("✅ Vue actualisée avec succès")
   } catch (error) {
     console.error("❌ Erreur lors de l'actualisation:", error)
     // Alerter l'équipe
   }
   ```

### ⚠️ À Éviter

1. **Ne pas confondre avec les collections**
   - Les vues normales sont read-only
   - Documentez clairement ce qui est une vue

2. **Éviter les vues sur des vues**
   ```javascript
   // ❌ DÉCONSEILLÉ
   db.createView("vue1", "collection", [...])
   db.createView("vue2", "vue1", [...])  // Vue sur une vue

   // ✅ MIEUX
   db.createView("vue2", "collection", [ /* pipeline combiné */ ])
   ```

3. **Ne pas oublier d'actualiser**
   - Les vues matérialisées deviennent obsolètes
   - Mettez en place un système d'actualisation fiable

4. **Attention aux pipelines coûteux**
   - Sur les vues normales, le pipeline s'exécute à chaque requête
   - Privilégiez les vues matérialisées pour les pipelines lourds

## Comparaison Finale

### Vue Normale vs Vue Matérialisée vs Collection

| Aspect | Vue Normale | Vue Matérialisée ($out/$merge) | Collection Normale |
|--------|-------------|--------------------------------|-------------------|
| **Stockage** | Définition uniquement | Données complètes | Données complètes |
| **Actualité** | Toujours à jour | Nécessite actualisation | Selon les écritures |
| **Performance lecture** | Variable (exécute pipeline) | Rapide (données stockées) | Rapide |
| **Écriture** | ❌ Impossible | ❌ Uniquement par pipeline | ✅ Oui |
| **Index** | Utilise ceux de la source | Peut avoir ses propres index | Oui |
| **Espace disque** | Minimal | Important | Important |
| **Complexité** | Simple | Moyenne (actualisation) | Simple |
| **Use case** | Requêtes fréquentes simples | Rapports complexes | Données transactionnelles |

## Résumé

### Vues Normales (Views)

**Quand utiliser :**
- ✅ Simplifier des requêtes complexes répétitives
- ✅ Masquer des données sensibles
- ✅ Créer des abstractions métier
- ✅ Les données doivent être toujours à jour

**Caractéristiques :**
- 🔄 Dynamiques (toujours à jour)
- 📖 Lecture seule
- 💾 Pas de stockage supplémentaire
- ⚡ Performance variable (exécute le pipeline)

### Vues Matérialisées

**Quand utiliser :**
- ✅ Rapports et dashboards
- ✅ Pipelines complexes et coûteux
- ✅ Données consultées fréquemment
- ✅ Actualité en temps réel pas nécessaire

**Caractéristiques :**
- 💾 Stocke les données
- ⚡ Performances optimales en lecture
- 🔄 Nécessite actualisation
- 📊 Peut avoir ses propres index

### Points Clés à Retenir

1. **Les vues simplifient le code**
   - Une définition, multiples utilisations
   - Code plus maintenable

2. **Choisir selon les besoins**
   - Vue normale : fraîcheur des données
   - Vue matérialisée : performance

3. **Optimiser les pipelines**
   - Les vues normales exécutent le pipeline à chaque fois
   - Suivre les bonnes pratiques d'optimisation (section 6.6)

4. **Planifier l'actualisation**
   - Les vues matérialisées doivent être actualisées
   - Automatiser avec des cron jobs ou Change Streams

5. **Documenter**
   - But de la vue
   - Fréquence d'actualisation (matérialisées)
   - Dépendances

---

**Règle d'or :**
> Utilisez les vues normales pour la simplicité et la fraîcheur, les vues matérialisées pour la performance et les analyses complexes.

Les vues sont un outil puissant pour organiser et optimiser l'accès à vos données MongoDB. Utilisées judicieusement, elles simplifient considérablement votre architecture et améliorent les performances de vos applications.

⏭️ [Validation des Schémas](/07-validation-des-schemas/README.md)
