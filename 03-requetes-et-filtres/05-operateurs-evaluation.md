🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 Opérateurs d'Évaluation

## Introduction

Les opérateurs que nous avons vus jusqu'à présent permettent de comparer des valeurs, de combiner des conditions logiques et de vérifier l'existence ou le type des champs. Cependant, certaines requêtes nécessitent des évaluations plus sophistiquées : rechercher des motifs dans du texte, comparer des champs entre eux, effectuer des calculs mathématiques, ou faire des recherches full-text.

MongoDB fournit des **opérateurs d'évaluation** qui permettent d'exécuter des opérations plus complexes lors des requêtes. Ces opérateurs offrent une grande flexibilité pour des cas d'usage avancés.

Dans ce chapitre, nous allons explorer cinq opérateurs d'évaluation principaux :
- **`$regex`** : recherche par expressions régulières
- **`$expr`** : évaluation d'expressions et comparaison de champs
- **`$mod`** : opération modulo
- **`$text`** : recherche full-text
- **`$where`** : exécution de code JavaScript (déprécié)

---

## Vue d'Ensemble des Opérateurs d'Évaluation

| Opérateur | Description | Usage Principal |
|-----------|-------------|-----------------|
| `$regex` | Recherche par expression régulière | Recherches de motifs dans du texte |
| `$expr` | Évaluation d'expressions agrégées | Comparaison de champs, calculs |
| `$mod` | Opération modulo | Filtrage basé sur division et reste |
| `$text` | Recherche full-text | Recherche de mots dans du texte indexé |
| `$where` | Exécution de JavaScript | Requêtes complexes (déprécié, éviter) |

---

## L'Opérateur `$regex`

L'opérateur `$regex` permet d'effectuer des recherches basées sur des **expressions régulières** (regex). C'est très utile pour rechercher des motifs dans du texte.

### Syntaxe

```javascript
{ champ: { $regex: /pattern/, $options: 'options' } }
// Ou
{ champ: { $regex: 'pattern', $options: 'options' } }
// Ou (syntaxe courte)
{ champ: /pattern/ }
```

### Expressions Régulières de Base

Avant d'utiliser `$regex`, voici quelques motifs de base :

| Motif | Signification | Exemple |
|-------|---------------|---------|
| `.` | N'importe quel caractère | `a.c` match "abc", "a1c" |
| `^` | Début de chaîne | `^Hello` match "Hello world" |
| `$` | Fin de chaîne | `world$` match "Hello world" |
| `*` | 0 ou plus occurrences | `ab*c` match "ac", "abc", "abbc" |
| `+` | 1 ou plus occurrences | `ab+c` match "abc", "abbc" |
| `?` | 0 ou 1 occurrence | `ab?c` match "ac", "abc" |
| `[]` | Classe de caractères | `[abc]` match "a", "b", ou "c" |
| `\|` | Ou logique | `cat\|dog` match "cat" ou "dog" |
| `()` | Groupe | `(ab)+` match "ab", "abab" |

### Options de `$regex`

| Option | Description |
|--------|-------------|
| `i` | Insensible à la casse (case-insensitive) |
| `m` | Multiline (^ et $ matchent chaque ligne) |
| `x` | Ignore les espaces blancs dans le pattern |
| `s` | Permet à `.` de matcher les nouvelles lignes |

### Exemples de Base

#### Recherche Simple

```javascript
// Trouver les utilisateurs dont le nom commence par "John"
db.users.find({ name: { $regex: /^John/ } })

// Trouver les emails se terminant par "@gmail.com"
db.users.find({ email: { $regex: /@gmail\.com$/ } })

// Trouver les produits contenant le mot "laptop"
db.products.find({ name: { $regex: /laptop/ } })
```

#### Recherche Insensible à la Casse

```javascript
// Trouver "john", "John", "JOHN", etc.
db.users.find({
    name: { $regex: /john/, $options: 'i' }
})

// Ou syntaxe alternative
db.users.find({
    name: { $regex: 'john', $options: 'i' }
})

// Trouver les produits contenant "laptop" (quelle que soit la casse)
db.products.find({
    name: { $regex: /laptop/i }
})
```

#### Recherche de Début de Mot

```javascript
// Noms commençant par "A"
db.users.find({ name: { $regex: /^A/ } })

// Emails du domaine example.com
db.users.find({ email: { $regex: /@example\.com$/ } })

// SKU commençant par "PROD"
db.products.find({ sku: { $regex: /^PROD/ } })
```

#### Recherche de Fin de Mot

```javascript
// Noms se terminant par "son"
db.users.find({ name: { $regex: /son$/ } })

// Fichiers avec extension .pdf
db.documents.find({ filename: { $regex: /\.pdf$/ } })

// URLs se terminant par .html
db.pages.find({ url: { $regex: /\.html$/ } })
```

#### Recherche Contenant

```javascript
// Descriptions contenant "urgent"
db.tasks.find({ description: { $regex: /urgent/i } })

// Produits avec "pro" dans le nom
db.products.find({ name: { $regex: /pro/i } })

// Articles contenant "MongoDB"
db.articles.find({ title: { $regex: /MongoDB/ } })
```

### Exemples Avancés

#### Alternance (OU)

```javascript
// Trouver "cat" OU "dog"
db.animals.find({ name: { $regex: /cat|dog/i } })

// Trouver les emails Gmail ou Yahoo
db.users.find({
    email: { $regex: /@(gmail|yahoo)\.com$/i }
})

// Trouver plusieurs mots-clés
db.products.find({
    description: { $regex: /laptop|computer|notebook/i }
})
```

#### Classes de Caractères

```javascript
// Noms commençant par une voyelle
db.users.find({ name: { $regex: /^[AEIOU]/i } })

// Codes commençant par un chiffre
db.products.find({ code: { $regex: /^[0-9]/ } })

// Codes alphanumériques
db.items.find({ code: { $regex: /^[A-Za-z0-9]+$/ } })
```

#### Quantificateurs

```javascript
// Numéros de téléphone (format: 10 chiffres)
db.users.find({ phone: { $regex: /^[0-9]{10}$/ } })

// Codes postaux français (5 chiffres)
db.addresses.find({ zipCode: { $regex: /^[0-9]{5}$/ } })

// Mots de 5 lettres ou plus
db.dictionary.find({ word: { $regex: /^[a-z]{5,}$/i } })
```

#### Groupes et Répétition

```javascript
// Mots répétés : "test test"
db.documents.find({
    content: { $regex: /\b(\w+)\s+\1\b/i }
})

// Dates format DD-MM-YYYY
db.events.find({
    date: { $regex: /^[0-3][0-9]-[0-1][0-9]-[0-9]{4}$/ }
})

// IPv4 (simple)
db.logs.find({
    ip: { $regex: /^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$/ }
})
```

### Cas d'Usage Pratiques

#### Recherche d'Utilisateurs

```javascript
// Recherche partielle de nom
db.users.find({
    name: { $regex: /john/i }
})

// Recherche par initiales
db.users.find({
    name: { $regex: /^J\.?D\.?/i }  // J.D. ou JD
})

// Emails d'un domaine spécifique
db.users.find({
    email: { $regex: /@company\.com$/i }
})

// Usernames alphanumériques uniquement
db.users.find({
    username: { $regex: /^[a-z0-9]+$/i }
})
```

#### Recherche de Produits

```javascript
// Produits avec "pro" ou "premium"
db.products.find({
    name: { $regex: /pro|premium/i }
})

// SKU format spécifique (ex: PROD-1234)
db.products.find({
    sku: { $regex: /^PROD-[0-9]{4}$/ }
})

// Descriptions contenant certains mots-clés
db.products.find({
    description: { $regex: /wireless|bluetooth|wifi/i }
})
```

#### Validation de Formats

```javascript
// URLs valides (simple)
db.links.find({
    url: { $regex: /^https?:\/\//i }
})

// Emails (pattern simple)
db.contacts.find({
    email: { $regex: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/ }
})

// Codes-barres (13 chiffres)
db.inventory.find({
    barcode: { $regex: /^[0-9]{13}$/ }
})
```

---

## L'Opérateur `$expr`

L'opérateur `$expr` permet d'utiliser des **expressions d'agrégation** dans les requêtes. Il est particulièrement utile pour **comparer des champs entre eux** ou effectuer des calculs.

### Syntaxe

```javascript
{ $expr: { expression_agregation } }
```

### Comparaison de Champs

L'utilisation principale de `$expr` est de comparer deux champs d'un même document.

#### Exemples de Base

```javascript
// Trouver les produits où le prix de vente est supérieur au coût
db.products.find({
    $expr: { $gt: ["$salePrice", "$cost"] }
})

// Trouver les commandes où la quantité livrée est inférieure à la quantité commandée
db.orders.find({
    $expr: { $lt: ["$deliveredQty", "$orderedQty"] }
})

// Trouver les utilisateurs où l'âge est égal aux points divisés par 10
db.users.find({
    $expr: { $eq: ["$age", { $divide: ["$points", 10] }] }
})
```

**Important** : Dans `$expr`, les noms de champs doivent être préfixés par `$`.

#### Opérateurs de Comparaison avec `$expr`

```javascript
// Égalité : prix de vente = coût
db.products.find({
    $expr: { $eq: ["$salePrice", "$cost"] }
})

// Différence : quantité en stock différente de la quantité minimale
db.inventory.find({
    $expr: { $ne: ["$currentStock", "$minStock"] }
})

// Supérieur : budget dépensé > budget alloué
db.projects.find({
    $expr: { $gt: ["$spent", "$budget"] }
})

// Supérieur ou égal : score >= score minimum requis
db.exams.find({
    $expr: { $gte: ["$score", "$minScore"] }
})

// Inférieur : stock actuel < seuil de réapprovisionnement
db.products.find({
    $expr: { $lt: ["$stock", "$reorderThreshold"] }
})

// Inférieur ou égal : prix <= budget maximum
db.products.find({
    $expr: { $lte: ["$price", "$maxBudget"] }
})
```

### Opérations Arithmétiques

`$expr` permet d'effectuer des calculs dans les requêtes :

```javascript
// Produits où le prix de vente est au moins le double du coût
db.products.find({
    $expr: {
        $gte: ["$salePrice", { $multiply: ["$cost", 2] }]
    }
})

// Utilisateurs où le score total est la somme de tous les scores partiels
db.users.find({
    $expr: {
        $eq: [
            "$totalScore",
            { $add: ["$score1", "$score2", "$score3"] }
        ]
    }
})

// Commandes avec une réduction de plus de 20%
db.orders.find({
    $expr: {
        $gt: [
            "$discount",
            { $multiply: ["$originalPrice", 0.20] }
        ]
    }
})
```

### Opérateurs Arithmétiques Disponibles

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$add` | Addition | `{ $add: ["$a", "$b"] }` |
| `$subtract` | Soustraction | `{ $subtract: ["$a", "$b"] }` |
| `$multiply` | Multiplication | `{ $multiply: ["$a", "$b"] }` |
| `$divide` | Division | `{ $divide: ["$a", "$b"] }` |
| `$mod` | Modulo | `{ $mod: ["$a", "$b"] }` |
| `$abs` | Valeur absolue | `{ $abs: "$value" }` |
| `$ceil` | Arrondi supérieur | `{ $ceil: "$value" }` |
| `$floor` | Arrondi inférieur | `{ $floor: "$value" }` |

### Opérations sur Chaînes

```javascript
// Utilisateurs où le nom complet = prénom + nom
db.users.find({
    $expr: {
        $eq: [
            "$fullName",
            { $concat: ["$firstName", " ", "$lastName"] }
        ]
    }
})

// Emails avec un domaine spécifique (en extrayant le domaine)
db.users.find({
    $expr: {
        $eq: [
            { $substr: ["$email", { $indexOfBytes: ["$email", "@"] }, -1] },
            "@company.com"
        ]
    }
})
```

### Opérations sur Tableaux

```javascript
// Documents où le nombre d'éléments dans le tableau = compteur
db.items.find({
    $expr: {
        $eq: ["$count", { $size: "$tags" }]
    }
})

// Produits où le nombre de reviews correspond au compteur
db.products.find({
    $expr: {
        $eq: [
            { $size: "$reviews" },
            "$reviewCount"
        ]
    }
})
```

### Opérations Conditionnelles

```javascript
// Utiliser $cond pour des conditions
db.products.find({
    $expr: {
        $gt: [
            {
                $cond: {
                    if: { $gte: ["$quantity", 10] },
                    then: { $multiply: ["$price", 0.9] },  // 10% de réduction
                    else: "$price"
                }
            },
            100
        ]
    }
})
```

### Cas d'Usage Pratiques

#### E-commerce : Analyse de Marges

```javascript
// Produits avec une marge bénéficiaire inférieure à 20%
db.products.find({
    $expr: {
        $lt: [
            { $subtract: ["$salePrice", "$cost"] },
            { $multiply: ["$cost", 0.20] }
        ]
    }
})

// Produits vendus à perte
db.products.find({
    $expr: { $lt: ["$salePrice", "$cost"] }
})

// Produits avec un prix de vente au moins 50% supérieur au coût
db.products.find({
    $expr: {
        $gte: [
            "$salePrice",
            { $multiply: ["$cost", 1.5] }
        ]
    }
})
```

#### Gestion de Stock

```javascript
// Produits nécessitant un réapprovisionnement
db.inventory.find({
    $expr: { $lt: ["$currentStock", "$minStock"] }
})

// Stock disponible insuffisant pour les commandes en attente
db.products.find({
    $expr: { $lt: ["$stock", "$pendingOrders"] }
})

// Produits en surstockage (2x le stock maximum)
db.inventory.find({
    $expr: { $gt: ["$currentStock", { $multiply: ["$maxStock", 2] }] }
})
```

#### Budgets et Finances

```javascript
// Projets dépassant leur budget
db.projects.find({
    $expr: { $gt: ["$spent", "$budget"] }
})

// Projets ayant dépensé plus de 80% du budget
db.projects.find({
    $expr: {
        $gte: [
            "$spent",
            { $multiply: ["$budget", 0.80] }
        ]
    }
})

// Budgets disponibles (budget - dépensé)
db.projects.find({
    $expr: {
        $gt: [
            { $subtract: ["$budget", "$spent"] },
            0
        ]
    }
})
```

---

## L'Opérateur `$mod`

L'opérateur `$mod` effectue une opération **modulo** et retourne les documents où le reste de la division correspond à une valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $mod: [diviseur, reste] } }
```

Cette requête retourne les documents où `champ % diviseur == reste`.

### Exemples de Base

```javascript
// Trouver les nombres pairs (reste 0 quand divisé par 2)
db.numbers.find({ value: { $mod: [2, 0] } })

// Trouver les nombres impairs (reste 1 quand divisé par 2)
db.numbers.find({ value: { $mod: [2, 1] } })

// Trouver les multiples de 5 (reste 0 quand divisé par 5)
db.numbers.find({ value: { $mod: [5, 0] } })

// Trouver les nombres qui donnent un reste de 3 quand divisés par 7
db.numbers.find({ value: { $mod: [7, 3] } })
```

### Cas d'Usage Pratiques

#### Pagination et Distribution

```javascript
// Distribuer les utilisateurs en 4 groupes (groupe 0)
db.users.find({ userId: { $mod: [4, 0] } })

// Utilisateurs du groupe 1
db.users.find({ userId: { $mod: [4, 1] } })

// Traiter une tâche toutes les 10 unités
db.tasks.find({ taskId: { $mod: [10, 0] } })
```

#### Séparation de Données

```javascript
// IDs pairs pour le serveur A
db.records.find({ id: { $mod: [2, 0] } })

// IDs impairs pour le serveur B
db.records.find({ id: { $mod: [2, 1] } })
```

#### Calendrier et Planification

```javascript
// Tâches planifiées tous les 3 jours (jour 0, 3, 6, 9, etc.)
db.tasks.find({ dayOfYear: { $mod: [3, 0] } })

// Événements ayant lieu toutes les 2 semaines
db.events.find({ weekNumber: { $mod: [2, 0] } })
```

#### Contrôle de Qualité

```javascript
// Sélectionner 1 produit sur 10 pour inspection
db.products.find({ serialNumber: { $mod: [10, 0] } })

// Échantillonnage : 1 commande sur 20
db.orders.find({ orderNumber: { $mod: [20, 0] } })
```

---

## L'Opérateur `$text`

L'opérateur `$text` permet d'effectuer des **recherches full-text** sur des champs indexés avec un index texte. C'est utile pour rechercher des mots dans du contenu textuel.

### Prérequis : Créer un Index Texte

Avant d'utiliser `$text`, vous devez créer un index texte :

```javascript
// Créer un index texte sur le champ description
db.products.createIndex({ description: "text" })

// Index texte sur plusieurs champs
db.articles.createIndex({
    title: "text",
    content: "text"
})

// Index texte avec pondération
db.articles.createIndex(
    {
        title: "text",
        content: "text"
    },
    {
        weights: {
            title: 10,     // Plus de poids au titre
            content: 1
        }
    }
)
```

### Syntaxe

```javascript
{
    $text: {
        $search: "terme de recherche",
        $language: "langue",         // Optionnel
        $caseSensitive: boolean,     // Optionnel
        $diacriticSensitive: boolean // Optionnel
    }
}
```

### Exemples de Base

```javascript
// Rechercher "mongodb"
db.articles.find({
    $text: { $search: "mongodb" }
})

// Rechercher plusieurs mots (OU implicite)
db.products.find({
    $text: { $search: "laptop computer" }
})
// Retourne documents contenant "laptop" OU "computer"

// Rechercher une phrase exacte
db.articles.find({
    $text: { $search: "\"MongoDB database\"" }
})
// Retourne documents contenant exactement "MongoDB database"
```

### Opérateurs de Recherche

#### Recherche de Plusieurs Mots (OU)

```javascript
// Rechercher "mongodb" OU "database"
db.articles.find({
    $text: { $search: "mongodb database" }
})
```

#### Phrase Exacte

```javascript
// Rechercher la phrase exacte entre guillemets
db.articles.find({
    $text: { $search: "\"NoSQL database\"" }
})
```

#### Exclusion de Mots

```javascript
// Rechercher "mongodb" mais PAS "tutorial"
db.articles.find({
    $text: { $search: "mongodb -tutorial" }
})

// Rechercher "database" mais PAS "SQL"
db.articles.find({
    $text: { $search: "database -SQL" }
})
```

### Score de Pertinence

MongoDB calcule un score de pertinence pour chaque résultat. Vous pouvez trier par ce score :

```javascript
// Rechercher et trier par pertinence
db.articles.find(
    { $text: { $search: "mongodb database" } },
    { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

### Langues Supportées

MongoDB supporte de nombreuses langues pour les recherches textuelles :

```javascript
// Recherche en français
db.articles.find({
    $text: {
        $search: "base de données",
        $language: "french"
    }
})

// Recherche en espagnol
db.articles.find({
    $text: {
        $search: "base de datos",
        $language: "spanish"
    }
})

// Langues supportées : english, french, german, spanish, italian,
// portuguese, russian, turkish, arabic, chinese, japanese, korean, etc.
```

### Options Avancées

```javascript
// Recherche sensible à la casse
db.articles.find({
    $text: {
        $search: "MongoDB",
        $caseSensitive: true
    }
})

// Recherche sensible aux accents
db.articles.find({
    $text: {
        $search: "café",
        $diacriticSensitive: true
    }
})
```

### Cas d'Usage Pratiques

#### Blog ou Site de Contenu

```javascript
// Créer l'index
db.articles.createIndex({ title: "text", content: "text" })

// Rechercher des articles
db.articles.find({
    $text: { $search: "MongoDB tutorial" }
})

// Rechercher avec score
db.articles.find(
    { $text: { $search: "database performance" } },
    { score: { $meta: "textScore" }, title: 1 }
).sort({ score: { $meta: "textScore" } }).limit(10)
```

#### E-commerce

```javascript
// Index sur nom et description
db.products.createIndex({
    name: "text",
    description: "text"
})

// Rechercher des produits
db.products.find({
    $text: { $search: "laptop gaming" }
})

// Exclure certains termes
db.products.find({
    $text: { $search: "phone -apple" }
})
```

#### Documentation ou Base de Connaissances

```javascript
// Index sur plusieurs champs
db.docs.createIndex({
    title: "text",
    summary: "text",
    content: "text"
})

// Recherche avec pondération
db.docs.find({
    $text: { $search: "authentication security" }
})
```

### Limitations de `$text`

- **Un seul index texte** par collection
- **Pas de regex** dans les recherches textuelles
- **Performance** : peut être lent sur de très grandes collections
- Pour des recherches plus avancées, considérez **Atlas Search**

---

## L'Opérateur `$where` (Déprécié)

L'opérateur `$where` permet d'exécuter du **code JavaScript** dans une requête.

### ⚠️ ATTENTION : Déprécié et Déconseillé

`$where` est **fortement déconseillé** pour les raisons suivantes :
- **Performance très faible** : nécessite l'exécution de JavaScript pour chaque document
- **Problèmes de sécurité** : risque d'injection de code
- **Pas d'index** : ne peut pas utiliser les index MongoDB

**Alternative recommandée** : Utilisez `$expr` à la place.

### Syntaxe (pour référence)

```javascript
{ $where: "code JavaScript" }
// Ou
{ $where: function() { return condition; } }
```

### Exemples (pour compréhension uniquement)

```javascript
// ❌ À ÉVITER : Utiliser $where
db.products.find({
    $where: "this.salePrice > this.cost * 2"
})

// ✅ PRÉFÉRER : Utiliser $expr
db.products.find({
    $expr: {
        $gt: ["$salePrice", { $multiply: ["$cost", 2] }]
    }
})
```

### Pourquoi `$where` Existe Encore

`$where` existe pour des raisons de compatibilité avec d'anciennes versions de MongoDB. Dans presque tous les cas, `$expr` est une meilleure alternative.

---

## Combinaison d'Opérateurs d'Évaluation

Vous pouvez combiner plusieurs opérateurs d'évaluation dans une même requête.

### Exemples de Combinaison

```javascript
// Regex + autres filtres
db.users.find({
    email: { $regex: /@company\.com$/i },
    status: "active",
    age: { $gte: 18 }
})

// $expr + filtres standards
db.products.find({
    category: "Electronics",
    $expr: { $gt: ["$salePrice", "$cost"] }
})

// $text + filtres
db.articles.find({
    $text: { $search: "mongodb tutorial" },
    status: "published",
    views: { $gte: 1000 }
})

// $mod + autres filtres
db.orders.find({
    orderNumber: { $mod: [10, 0] },
    status: "completed",
    amount: { $gte: 100 }
})
```

### Exemple Complexe

```javascript
// Recherche avancée de produits
db.products.find({
    // Recherche textuelle
    $text: { $search: "laptop gaming" },

    // Regex sur le SKU
    sku: { $regex: /^TECH-/i },

    // Comparaison de champs
    $expr: {
        $gte: ["$salePrice", { $multiply: ["$cost", 1.2] }]
    },

    // Filtres standards
    stock: { $gt: 0 },
    rating: { $gte: 4.0 }
})
```

---

## Comparaison des Opérateurs

| Besoin | Opérateur Recommandé | Alternative |
|--------|---------------------|-------------|
| Recherche de motif dans texte | `$regex` | `$text` (pour mots) |
| Comparaison de champs | `$expr` | N/A |
| Recherche full-text | `$text` | `$regex` (limité) |
| Opération modulo | `$mod` | `$expr` avec `$mod` |
| Logique complexe | `$expr` | **Jamais** `$where` |

---

## Bonnes Pratiques

### 1. Préférer les Index pour `$regex`

```javascript
// Créer un index pour les requêtes regex fréquentes
db.users.createIndex({ email: 1 })

// Regex optimisée (commence par ^)
db.users.find({ email: { $regex: /^john/i } })
```

### 2. Ancrer les Regex Quand Possible

```javascript
// ✅ Bon : regex ancrée au début (peut utiliser un index)
db.users.find({ name: { $regex: /^John/i } })

// ⚠️ Moins optimal : recherche au milieu (scan complet)
db.users.find({ name: { $regex: /John/i } })
```

### 3. Utiliser `$expr` au Lieu de `$where`

```javascript
// ❌ À éviter
db.products.find({
    $where: "this.price > this.cost"
})

// ✅ Préférer
db.products.find({
    $expr: { $gt: ["$price", "$cost"] }
})
```

### 4. Créer des Index Texte pour `$text`

```javascript
// Toujours créer un index texte avant d'utiliser $text
db.articles.createIndex({
    title: "text",
    content: "text"
})
```

### 5. Combiner Filtres Sélectifs en Premier

```javascript
// ✅ Bon : filtre sélectif d'abord
db.products.find({
    category: "Electronics",  // Très sélectif
    $text: { $search: "laptop" }
})

// ⚠️ Moins optimal : recherche texte d'abord
db.products.find({
    $text: { $search: "laptop" },
    category: "Electronics"
})
```

### 6. Limiter l'Utilisation de Regex Complexes

```javascript
// ✅ Simple et efficace
db.users.find({ email: { $regex: /@gmail\.com$/i } })

// ⚠️ Complexe et lent
db.users.find({
    email: {
        $regex: /^(?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\])$/i
    }
})
```

---

## Performance et Optimisation

### Impact des Opérateurs

| Opérateur | Performance | Peut Utiliser Index | Notes |
|-----------|-------------|---------------------|-------|
| `$regex` | Moyenne | Partiel (si ^) | Ancrer au début améliore la performance |
| `$expr` | Moyenne | Non directement | Calculs peuvent être coûteux |
| `$mod` | Bonne | Oui | Simple et rapide |
| `$text` | Moyenne | Oui (index texte) | Nécessite un index texte |
| `$where` | Très Faible | Non | **À ÉVITER** |

### Optimisation de `$regex`

```javascript
// ✅ Peut utiliser un index
db.users.find({ name: { $regex: /^John/i } })

// ❌ Ne peut pas utiliser d'index efficacement
db.users.find({ name: { $regex: /John/i } })
```

### Optimisation de `$expr`

```javascript
// Combiner avec des filtres indexés
db.products.find({
    category: "Electronics",  // Utilise l'index
    $expr: { $gt: ["$salePrice", "$cost"] }
})
```

### Vérification avec `explain()`

```javascript
// Analyser une requête regex
db.users.find({
    email: { $regex: /^john/i }
}).explain("executionStats")

// Analyser une requête avec $expr
db.products.find({
    $expr: { $gt: ["$salePrice", "$cost"] }
}).explain("executionStats")

// Analyser une recherche texte
db.articles.find({
    $text: { $search: "mongodb" }
}).explain("executionStats")
```

---

## Points Clés à Retenir

✅ **`$regex`** permet des recherches par motifs avec expressions régulières

✅ Ancrer les regex avec **`^`** améliore les performances et permet l'utilisation d'index

✅ **`$expr`** permet de comparer des champs et d'effectuer des calculs dans les requêtes

✅ Les champs dans **`$expr`** doivent être préfixés par **`$`**

✅ **`$mod`** effectue des opérations modulo pour filtrer par reste de division

✅ **`$text`** nécessite un **index texte** préalable

✅ **`$text`** supporte les recherches de mots, phrases exactes et exclusions

✅ **`$where`** est **déprécié** et doit être remplacé par **`$expr`**

✅ Les opérateurs d'évaluation peuvent être **combinés** avec d'autres opérateurs

✅ Utilisez **`explain()`** pour optimiser les requêtes complexes

---

## Résumé

Dans ce chapitre, vous avez appris :

- Comment utiliser `$regex` pour des recherches par motifs avec expressions régulières
- Comment utiliser `$expr` pour comparer des champs et effectuer des calculs
- Comment utiliser `$mod` pour des opérations modulo
- Comment utiliser `$text` pour des recherches full-text
- Pourquoi éviter `$where` et utiliser `$expr` à la place
- Comment combiner ces opérateurs pour des requêtes sophistiquées
- Les bonnes pratiques d'optimisation et de performance
- L'importance des index pour chaque opérateur

Ces opérateurs d'évaluation complètent votre arsenal de requêtage MongoDB et vous permettent de créer des requêtes très sophistiquées pour répondre à des besoins complexes.

Dans le prochain chapitre, nous explorerons les **opérateurs de tableaux** qui vous permettront de travailler efficacement avec des données structurées en tableaux.

---


⏭️ [Opérateurs de tableaux ($all, $elemMatch, $size)](/03-requetes-et-filtres/06-operateurs-tableaux.md)
