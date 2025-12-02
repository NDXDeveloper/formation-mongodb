🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 Niveaux de validation (strict, moderate)

## 📚 Vue d'ensemble

Les **niveaux de validation** (`validationLevel`) déterminent **quels documents** MongoDB doit valider dans une collection. C'est comme choisir si vous voulez appliquer de nouvelles règles à tout le monde ou seulement aux nouveaux arrivants.

MongoDB propose deux niveaux de validation :
- **`strict`** : Valide tous les documents (par défaut)
- **`moderate`** : Valide uniquement les nouveaux documents

---

## 🤔 Pourquoi deux niveaux ?

### Le problème des collections existantes

Imaginez cette situation :

1. Vous avez une collection `utilisateurs` avec **10 000 documents** déjà existants
2. Ces documents ont été créés **sans validation** pendant 2 ans
3. Ils contiennent des **incohérences** : champs manquants, types incorrects, formats variables
4. Vous voulez maintenant **ajouter de la validation** pour améliorer la qualité

**Dilemme** :
- ❓ Si vous validez tout : vos 10 000 documents existants risquent d'être **non conformes**
- ❓ Vous ne pouvez plus les modifier sans les corriger d'abord
- ❓ Cela peut **bloquer** votre application

**Solution** : Le niveau `moderate` !

### Analogie avec les règles de copropriété

**Niveau strict** = Nouvelle règle qui s'applique à **tous les copropriétaires** (anciens et nouveaux)
- Si vous ne respectez pas, vous ne pouvez rien faire
- Tout le monde doit se mettre en conformité immédiatement

**Niveau moderate** = Nouvelle règle qui s'applique uniquement aux **nouveaux copropriétaires**
- Les anciens conservent leurs habitudes
- Les nouveaux doivent respecter les nouvelles règles
- Migration progressive et en douceur

---

## 🎯 Niveau `strict` (par défaut)

### Définition

Le niveau **strict** valide **TOUS** les documents :
- ✅ Les **insertions** de nouveaux documents
- ✅ Les **modifications** de documents existants (même non conformes au départ)

### Comportement

```javascript
db.createCollection("produits", {
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "strict"  // Par défaut, peut être omis
})
```

**Règle** : Dès qu'un document est touché (insertion ou modification), il DOIT être conforme.

### Exemple pratique

```javascript
// Créer une collection avec validation stricte
db.createCollection("employes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email", "age"],
      properties: {
        nom: { bsonType: "string", minLength: 2 },
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        },
        age: {
          bsonType: "int",
          minimum: 18,
          maximum: 65
        }
      }
    }
  },
  validationLevel: "strict"
})

// ✅ Insertion valide
db.employes.insertOne({
  nom: "Dupont",
  email: "dupont@example.com",
  age: 30
})

// ❌ Insertion invalide : email manquant
db.employes.insertOne({
  nom: "Martin",
  age: 25
})
// Erreur : Document failed validation

// Supposons qu'un document non conforme existe déjà (ajouté avant la validation)
// { _id: 1, nom: "Ancien", age: "35" }  // age est un string au lieu de int

// ❌ Modification impossible : le document doit devenir conforme
db.employes.updateOne(
  { _id: 1 },
  { $set: { nom: "Ancien Modifié" } }
)
// Erreur : Document failed validation
// Car "age" reste un string et doit être un int
```

### Quand utiliser le niveau strict

- ✅ **Nouvelles collections** - Collections créées avec validation dès le départ
- ✅ **Données propres** - Vous êtes sûr que tous vos documents sont conformes
- ✅ **Environnements de développement** - Pour garantir la qualité dès le début
- ✅ **Applications critiques** - Quand la cohérence est essentielle
- ✅ **Après nettoyage** - Quand vous avez corrigé tous les documents non conformes

### Avantages et inconvénients

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Cohérence maximale | ❌ Peut bloquer les opérations sur documents existants |
| ✅ Garantie de qualité | ❌ Nécessite un nettoyage préalable des données |
| ✅ Détection immédiate des problèmes | ❌ Migration difficile sur collections existantes |
| ✅ Simplicité conceptuelle | ❌ Peut casser les applications existantes |

---

## 📦 Niveau `moderate`

### Définition

Le niveau **moderate** valide **SEULEMENT** :
- ✅ Les **insertions** de nouveaux documents
- ❌ Les **modifications** de documents existants sont **exemptées** de validation

**Exception** : Si vous modifiez un document existant **qui était déjà conforme**, il doit rester conforme.

### Comportement

```javascript
db.createCollection("produits", {
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "moderate"
})
```

**Règle** :
- Nouveaux documents → **Validés**
- Documents existants non conformes → **Peuvent être modifiés librement**
- Documents existants conformes → **Doivent rester conformes**

### Exemple pratique

```javascript
// Créer une collection avec validation modérée
db.createCollection("clients", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string" },
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        }
      }
    }
  },
  validationLevel: "moderate"
})

// ✅ Insertion valide
db.clients.insertOne({
  nom: "Bernard",
  email: "bernard@example.com"
})

// ❌ Insertion invalide : email manquant
db.clients.insertOne({
  nom: "Leroux"
})
// Erreur : Document failed validation

// Supposons que ces documents existent déjà (créés avant la validation) :
// Document 1 : { _id: 1, nom: "Ancien Client" }  // Non conforme (email manquant)
// Document 2 : { _id: 2, nom: "Bon Client", email: "bon@example.com" }  // Conforme

// ✅ Modification autorisée : document non conforme peut être modifié
db.clients.updateOne(
  { _id: 1 },
  { $set: { nom: "Ancien Client Modifié" } }
)
// Succès ! Pas de validation car document était déjà non conforme

// ✅ Modification autorisée : document conforme reste conforme
db.clients.updateOne(
  { _id: 2 },
  { $set: { nom: "Très Bon Client" } }
)
// Succès ! Le document reste valide

// ❌ Modification refusée : document conforme ne doit pas devenir non conforme
db.clients.updateOne(
  { _id: 2 },
  { $unset: { email: "" } }  // Retire l'email
)
// Erreur : Document failed validation
// Car le document était conforme et ne peut pas devenir non conforme
```

### Quand utiliser le niveau moderate

- ✅ **Collections existantes** - Avec des données historiques non conformes
- ✅ **Migration progressive** - Transition vers la validation en douceur
- ✅ **Refonte de schéma** - Changement de structure sur données existantes
- ✅ **Environnements de production** - Pour ne pas casser l'existant
- ✅ **Phase de test** - Tester la validation sans impacter l'ancien code

### Avantages et inconvénients

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Pas de blocage sur données existantes | ❌ Cohérence partielle (mix ancien/nouveau) |
| ✅ Migration progressive possible | ❌ Complexité conceptuelle |
| ✅ Pas de nettoyage préalable obligatoire | ❌ Peut créer de la confusion |
| ✅ Sécurise l'avenir sans casser le passé | ❌ Documents incohérents peuvent persister |

---

## 🔄 Comparaison des deux niveaux

### Tableau comparatif

| Opération | Document existant | Niveau `strict` | Niveau `moderate` |
|-----------|-------------------|-----------------|-------------------|
| **INSERT** nouveau | N/A | ✅ Validé | ✅ Validé |
| **UPDATE** document non conforme | Oui | ❌ Refusé | ✅ Autorisé |
| **UPDATE** document conforme → conforme | Oui | ✅ Autorisé | ✅ Autorisé |
| **UPDATE** document conforme → non conforme | Oui | ❌ Refusé | ❌ Refusé |

### Schéma visuel

```
NIVEAU STRICT
═════════════

Nouveau document          → [VALIDATION] → ✅ ou ❌
Document existant modifié → [VALIDATION] → ✅ ou ❌
Tous les documents sont validés sans exception


NIVEAU MODERATE
═══════════════

Nouveau document                        → [VALIDATION] → ✅ ou ❌
Document existant NON conforme modifié  → [PAS DE VALIDATION] → ✅
Document existant conforme modifié      → [VALIDATION] → ✅ ou ❌
Seuls les nouveaux et les conformes sont validés
```

### Exemple comparatif complet

```javascript
// Collection avec documents existants non conformes
// { _id: 1, nom: "User1" }  // Manque "email"
// { _id: 2, nom: "User2", email: "user2@example.com" }  // Conforme

// Schéma de validation
const validationRules = {
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email"],
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" }
    }
  }
}

// ────────────────────────────────────────────
// SCÉNARIO 1 : NIVEAU STRICT
// ────────────────────────────────────────────

db.createCollection("users_strict", {
  validator: validationRules,
  validationLevel: "strict"
})

// ✅ Nouvel insert valide
db.users_strict.insertOne({ nom: "User3", email: "user3@example.com" })

// ❌ Modification du document non conforme refusée
db.users_strict.updateOne(
  { _id: 1 },
  { $set: { nom: "User1 Modified" } }
)
// Erreur : Document failed validation

// ❌ Modification du document conforme vers non conforme refusée
db.users_strict.updateOne(
  { _id: 2 },
  { $unset: { email: "" } }
)
// Erreur : Document failed validation


// ────────────────────────────────────────────
// SCÉNARIO 2 : NIVEAU MODERATE
// ────────────────────────────────────────────

db.createCollection("users_moderate", {
  validator: validationRules,
  validationLevel: "moderate"
})

// ✅ Nouvel insert valide
db.users_moderate.insertOne({ nom: "User3", email: "user3@example.com" })

// ✅ Modification du document non conforme autorisée
db.users_moderate.updateOne(
  { _id: 1 },
  { $set: { nom: "User1 Modified" } }
)
// Succès ! Pas de validation

// ❌ Modification du document conforme vers non conforme refusée
db.users_moderate.updateOne(
  { _id: 2 },
  { $unset: { email: "" } }
)
// Erreur : Document failed validation
```

---

## 🔧 Changer le niveau de validation

### Sur une collection existante

Utilisez la commande `collMod` :

```javascript
// Passer en mode strict
db.runCommand({
  collMod: "maCollection",
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "strict"
})

// Passer en mode moderate
db.runCommand({
  collMod: "maCollection",
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "moderate"
})
```

### Consulter le niveau actuel

```javascript
// Voir la configuration de validation
db.getCollectionInfos({ name: "maCollection" })[0].options.validationLevel
```

### Exemple de modification

```javascript
// État initial : strict
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix"],
      properties: {
        nom: { bsonType: "string" },
        prix: { bsonType: "double", minimum: 0 }
      }
    }
  },
  validationLevel: "strict"
})

// Vérifier le niveau
db.getCollectionInfos({ name: "produits" })[0].options.validationLevel
// Résultat : "strict"

// Passer en moderate
db.runCommand({
  collMod: "produits",
  validationLevel: "moderate"
})

// Vérifier à nouveau
db.getCollectionInfos({ name: "produits" })[0].options.validationLevel
// Résultat : "moderate"
```

---

## 📋 Stratégies de migration

### Stratégie 1 : De `moderate` vers `strict` (recommandé)

**Étapes** :

1. **Activer la validation en mode `moderate`**
```javascript
db.createCollection("users", {
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "moderate"
})
```

2. **Laisser l'application fonctionner normalement**
   - Les nouveaux documents sont conformes
   - Les anciens peuvent encore être modifiés

3. **Identifier les documents non conformes**
```javascript
// Script pour trouver les documents non conformes
db.users.find({
  $or: [
    { email: { $exists: false } },
    { email: { $not: { $regex: "^.+@.+\\..+$" } } }
  ]
})
```

4. **Corriger les documents non conformes**
```javascript
// Exemple : ajouter un email par défaut
db.users.updateMany(
  { email: { $exists: false } },
  { $set: { email: "noemail@example.com" } }
)
```

5. **Passer en mode `strict`**
```javascript
db.runCommand({
  collMod: "users",
  validationLevel: "strict"
})
```

6. **Tester que tout fonctionne**

### Stratégie 2 : Nettoyage avant validation `strict`

Si vous avez peu de documents ou un projet nouveau :

1. **Nettoyer d'abord les données**
```javascript
// Supprimer ou corriger les documents invalides
db.users.deleteMany({ email: { $exists: false } })
```

2. **Activer directement en mode `strict`**
```javascript
db.createCollection("users", {
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "strict"
})
```

### Stratégie 3 : Double collection (pour grandes migrations)

Pour les très grandes collections :

1. **Créer une nouvelle collection avec validation**
```javascript
db.createCollection("users_v2", {
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "strict"
})
```

2. **Migrer progressivement les données**
```javascript
// Pipeline d'agrégation pour nettoyer et migrer
db.users.aggregate([
  {
    $match: {
      email: { $exists: true, $regex: "^.+@.+\\..+$" }
    }
  },
  {
    $out: "users_v2"
  }
])
```

3. **Basculer l'application vers la nouvelle collection**

4. **Supprimer l'ancienne collection après validation**

---

## 🎯 Cas d'usage réels

### Cas 1 : Startup en croissance

**Situation** :
- Application lancée rapidement sans validation
- 50 000 utilisateurs avec données incohérentes
- Besoin d'améliorer la qualité des données

**Solution** :
```javascript
// 1. Activer validation moderate
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "nom"],
      properties: {
        email: { bsonType: "string", pattern: "^.+@.+\\..+$" },
        nom: { bsonType: "string", minLength: 2 }
      }
    }
  },
  validationLevel: "moderate"
})

// 2. Tous les nouveaux utilisateurs sont conformes
// 3. Migration progressive des anciens utilisateurs en arrière-plan
// 4. Passage en strict une fois tout nettoyé
```

### Cas 2 : Changement de modèle de données

**Situation** :
- Changement de structure : `telephone` devient `contacts.telephone`
- 100 000 documents avec ancienne structure
- Application doit continuer de fonctionner

**Solution** :
```javascript
// 1. Nouveau schéma avec validation moderate
db.runCommand({
  collMod: "clients",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        contacts: {
          bsonType: "object",
          properties: {
            telephone: { bsonType: "string" },
            email: { bsonType: "string" }
          }
        }
      }
    }
  },
  validationLevel: "moderate"
})

// 2. Application gère les deux structures
// 3. Migration progressive en arrière-plan
db.clients.updateMany(
  { telephone: { $exists: true } },
  [{
    $set: {
      contacts: {
        telephone: "$telephone",
        email: "$email"
      }
    }
  }]
)

// 4. Suppression des anciens champs
db.clients.updateMany(
  { telephone: { $exists: true } },
  { $unset: { telephone: "", email: "" } }
)

// 5. Passage en strict
```

### Cas 3 : Application critique en production

**Situation** :
- Application bancaire avec données sensibles
- Besoin de validation stricte dès le départ
- Pas de données existantes non conformes

**Solution** :
```javascript
// Validation strict dès la création
db.createCollection("transactions", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["montant", "date", "compteSource", "compteDestination"],
      properties: {
        montant: {
          bsonType: "decimal",
          minimum: 0.01,
          description: "Montant positif requis"
        },
        date: {
          bsonType: "date",
          description: "Date de la transaction"
        },
        compteSource: {
          bsonType: "string",
          pattern: "^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$",
          description: "IBAN valide"
        },
        compteDestination: {
          bsonType: "string",
          pattern: "^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$",
          description: "IBAN valide"
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})
```

---

## 💡 Conseils pratiques

### 1. Commencez toujours par `moderate` sur l'existant

Sur des collections existantes, commencez par `moderate` pour éviter les mauvaises surprises.

```javascript
// ✅ Approche sûre
db.runCommand({
  collMod: "ancienneCollection",
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "moderate",
  validationAction: "warn"  // En plus, mode warn au début
})
```

### 2. Utilisez `strict` uniquement quand vous êtes sûr

Passez en `strict` seulement après avoir :
- ✅ Testé la validation en `moderate`
- ✅ Identifié et corrigé les documents non conformes
- ✅ Vérifié que l'application fonctionne correctement

### 3. Documentez votre choix

Ajoutez des commentaires expliquant pourquoi vous avez choisi un niveau :

```javascript
// Niveau moderate car migration en cours depuis l'ancienne structure
// TODO: Passer en strict après nettoyage complet (ticket #1234)
db.runCommand({
  collMod: "produits",
  validationLevel: "moderate"
})
```

### 4. Combinez avec `validationAction`

Pour une migration encore plus douce :

```javascript
// Phase 1 : Observe sans bloquer
{
  validationLevel: "moderate",
  validationAction: "warn"
}

// Phase 2 : Valide les nouveaux sans bloquer l'ancien
{
  validationLevel: "moderate",
  validationAction: "error"
}

// Phase 3 : Valide tout
{
  validationLevel: "strict",
  validationAction: "error"
}
```

### 5. Automatisez les vérifications

Créez des scripts pour vérifier la conformité :

```javascript
// Script de vérification
function checkConformance(collectionName, validator) {
  const total = db[collectionName].countDocuments()

  // Essayer de valider chaque document
  const nonConformCount = db[collectionName].aggregate([
    {
      $match: {
        $nor: [validator.$jsonSchema]
      }
    },
    {
      $count: "total"
    }
  ]).toArray()[0]?.total || 0

  console.log(`Total documents: ${total}`)
  console.log(`Non-conformes: ${nonConformCount}`)
  console.log(`Conformité: ${((total - nonConformCount) / total * 100).toFixed(2)}%`)
}
```

---

## ⚠️ Pièges à éviter

### 1. Passer en `strict` sans tester

```javascript
// ❌ DANGER : Peut casser l'application
db.runCommand({
  collMod: "importanteCollection",
  validationLevel: "strict"  // Sans avoir testé avant !
})
```

### 2. Oublier que `moderate` valide les documents conformes

```javascript
// Document existant conforme :
// { _id: 1, nom: "User", email: "user@example.com" }

// ❌ Cette modification sera refusée même en moderate
db.users.updateOne(
  { _id: 1 },
  { $unset: { email: "" } }  // Rendre non conforme
)
// Car le document ÉTAIT conforme
```

### 3. Ne pas documenter le niveau choisi

Sans documentation, les autres développeurs ne comprennent pas pourquoi `moderate` est utilisé.

### 4. Garder `moderate` indéfiniment

`moderate` est un état de transition, pas une solution permanente.

```javascript
// ❌ Mauvaise pratique : moderate depuis 2 ans
// ✅ Bonne pratique : moderate temporaire avec plan de migration
```

---

## 🎓 Résumé

| Aspect | `strict` | `moderate` |
|--------|----------|------------|
| **Nouveaux documents** | ✅ Validés | ✅ Validés |
| **Docs existants non conformes** | ❌ Bloqués si modifiés | ✅ Peuvent être modifiés |
| **Docs existants conformes** | ✅ Doivent rester conformes | ✅ Doivent rester conformes |
| **Cas d'usage** | Collections neuves ou nettoyées | Collections existantes avec historique |
| **Niveau de garantie** | Maximum (100% conforme) | Partiel (nouveaux conformes) |
| **Risque de blocage** | Élevé | Faible |
| **Recommandé pour** | Production après tests | Migration progressive |

### Points clés à retenir

- ✅ **`strict`** = Tous les documents sont validés (nouveau par défaut)
- ✅ **`moderate`** = Seuls les nouveaux documents sont validés
- ✅ Utilisez `moderate` pour les **migrations** sur données existantes
- ✅ Passez en `strict` après **nettoyage** et **tests**
- ✅ Combinez avec `validationAction` pour plus de souplesse
- ✅ Documentez votre choix et planifiez la migration

---

## 📚 Dans la prochaine section

Dans la section suivante (7.5), nous verrons les **actions de validation** (`error` vs `warn`) qui déterminent ce qui se passe quand un document ne respecte pas les règles.

---


⏭️ [Actions de validation (error, warn)](/07-validation-des-schemas/05-actions-validation.md)
