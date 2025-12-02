🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.6 Modification des règles de validation

## 📚 Vue d'ensemble

La **modification des règles de validation** est une opération courante dans le cycle de vie d'une application. Vos besoins évoluent, votre modèle de données change, et vous devez adapter les règles de validation en conséquence.

Dans cette section, nous verrons comment modifier les règles de validation de manière **sûre et progressive**, sans casser votre application existante.

---

## 🤔 Pourquoi modifier les règles de validation ?

### Évolutions courantes

**1. Ajout de nouvelles contraintes**
- Un champ facultatif devient obligatoire
- Ajout de limites min/max sur les valeurs
- Ajout de formats spécifiques (regex)

**2. Assouplissement des règles**
- Un champ obligatoire devient facultatif
- Élargissement des valeurs autorisées
- Retrait de contraintes trop strictes

**3. Refonte du modèle de données**
- Changement de structure des documents
- Ajout ou suppression de champs
- Renommage de propriétés

**4. Correction d'erreurs**
- Règles trop strictes qui bloquent des cas légitimes
- Règles incorrectes ou incohérentes
- Bugs dans les expressions régulières

### Analogie avec un règlement

Imaginez un règlement de copropriété :

**Modifier les règles** = Voter un amendement au règlement
- Peut rendre les règles plus strictes (ex: interdire quelque chose)
- Peut les assouplir (ex: autoriser quelque chose)
- Doit être fait avec précaution pour ne pas créer de chaos
- Nécessite une transition si impact important

---

## 🔧 Commande de modification : `collMod`

### Syntaxe de base

La commande `collMod` (pour "Collection Modifier") permet de modifier les propriétés d'une collection, y compris ses règles de validation.

```javascript
db.runCommand({
  collMod: "nomDeLaCollection",
  validator: {
    $jsonSchema: {
      // Nouvelles règles complètes
    }
  },
  validationLevel: "strict",  // ou "moderate"
  validationAction: "error"   // ou "warn"
})
```

**Important** : `collMod` **remplace complètement** les règles existantes. Vous devez fournir le schéma complet, pas seulement les modifications.

### Exemple simple

```javascript
// Collection initiale avec validation simple
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom"],
      properties: {
        nom: { bsonType: "string" }
      }
    }
  }
})

// Modifier pour ajouter une contrainte sur le prix
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix"],  // Ajout de "prix" comme requis
      properties: {
        nom: { bsonType: "string" },
        prix: {  // Nouvelle propriété
          bsonType: "double",
          minimum: 0
        }
      }
    }
  }
})
```

---

## 🔍 Consulter les règles actuelles

### Avant de modifier

Avant de modifier les règles, il est essentiel de **consulter les règles actuelles** pour ne rien oublier.

```javascript
// Récupérer toutes les informations de la collection
db.getCollectionInfos({ name: "produits" })

// Résultat :
[
  {
    name: "produits",
    type: "collection",
    options: {
      validator: {
        $jsonSchema: {
          bsonType: "object",
          required: ["nom"],
          properties: {
            nom: { bsonType: "string" }
          }
        }
      },
      validationLevel: "strict",
      validationAction: "error"
    },
    info: {
      readOnly: false
    }
  }
]
```

### Extraire uniquement le validator

```javascript
// Récupérer uniquement les règles de validation
const collInfo = db.getCollectionInfos({ name: "produits" })[0]
const currentValidator = collInfo.options.validator
const currentLevel = collInfo.options.validationLevel
const currentAction = collInfo.options.validationAction

print("Validator actuel:")
printjson(currentValidator)
print("Level:", currentLevel)
print("Action:", currentAction)
```

### Script helper pour voir la validation

```javascript
// Fonction utilitaire pour afficher la validation actuelle
function showValidation(collectionName) {
  const info = db.getCollectionInfos({ name: collectionName })[0]

  if (!info) {
    print(`Collection "${collectionName}" n'existe pas`)
    return
  }

  if (!info.options.validator) {
    print(`Collection "${collectionName}" n'a pas de validation`)
    return
  }

  print(`\n=== Validation de "${collectionName}" ===`)
  print("\nValidator:")
  printjson(info.options.validator)
  print("\nValidation Level:", info.options.validationLevel || "strict (défaut)")
  print("Validation Action:", info.options.validationAction || "error (défaut)")
}

// Utilisation
showValidation("produits")
```

---

## ➕ Ajouter de nouvelles règles

### Scénario : Ajouter un champ obligatoire

**Situation** : Vous voulez rendre le champ `email` obligatoire.

**Étape 1 : Récupérer les règles actuelles**

```javascript
const collInfo = db.getCollectionInfos({ name: "utilisateurs" })[0]
const currentValidator = collInfo.options.validator
printjson(currentValidator)

// Règles actuelles :
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom"],
    properties: {
      nom: { bsonType: "string" }
    }
  }
}
```

**Étape 2 : Modifier en ajoutant la nouvelle règle**

```javascript
// ATTENTION : On doit TOUT réécrire, pas juste ajouter
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],  // Ajout d'email
      properties: {
        nom: { bsonType: "string" },
        email: {  // Nouvelle propriété
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        }
      }
    }
  }
})
```

**Important** : Si vous avez des documents existants sans `email`, cette modification en mode `strict` les rendra non modifiables !

**Solution sécurisée** : Passer d'abord en mode `moderate` ou `warn`

```javascript
// Option 1 : Mode moderate (nouveaux documents uniquement)
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string", pattern: "^.+@.+\\..+$" }
      }
    }
  },
  validationLevel: "moderate"  // Les anciens documents sans email peuvent être modifiés
})

// Option 2 : Mode warn (observer l'impact)
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string", pattern: "^.+@.+\\..+$" }
      }
    }
  },
  validationAction: "warn"  // Accepte tout mais log les violations
})
```

### Scénario : Ajouter des contraintes sur un champ existant

**Situation** : Le champ `age` existe mais sans contrainte. On veut ajouter des limites.

```javascript
// Avant : age sans contrainte
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      nom: { bsonType: "string" },
      age: { bsonType: "int" }  // Pas de min/max
    }
  }
}

// Après : age avec contraintes
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        nom: { bsonType: "string" },
        age: {
          bsonType: "int",
          minimum: 0,      // Nouveau
          maximum: 150     // Nouveau
        }
      }
    }
  },
  validationAction: "warn"  // Observer d'abord l'impact
})

// Vérifier combien de documents seraient affectés
db.utilisateurs.countDocuments({
  age: { $exists: true, $or: [{ $lt: 0 }, { $gt: 150 }] }
})
```

---

## ➖ Assouplir ou retirer des règles

### Scénario : Rendre un champ facultatif

**Situation** : Le champ `telephone` était obligatoire, mais vous voulez le rendre facultatif.

```javascript
// Avant : telephone requis
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email", "telephone"],
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" },
      telephone: { bsonType: "string" }
    }
  }
}

// Après : telephone facultatif
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],  // telephone retiré
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string" },
        telephone: { bsonType: "string" }  // Reste défini mais pas requis
      }
    }
  }
})
```

**Impact** : Assouplir les règles est généralement **sans danger** car cela rend plus de documents valides.

### Scénario : Élargir les valeurs autorisées

**Situation** : Ajouter une nouvelle valeur à un `enum`.

```javascript
// Avant : 3 statuts possibles
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      statut: {
        enum: ["actif", "inactif", "suspendu"]
      }
    }
  }
}

// Après : ajout de "archive"
db.runCommand({
  collMod: "comptes",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        statut: {
          enum: ["actif", "inactif", "suspendu", "archive"]  // Nouveau
        }
      }
    }
  }
})
```

### Scénario : Retirer complètement une contrainte

**Situation** : Retirer la contrainte de longueur minimale sur un champ.

```javascript
// Avant : nom avec minLength
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      nom: {
        bsonType: "string",
        minLength: 5  // Trop strict !
      }
    }
  }
}

// Après : pas de minLength
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        nom: {
          bsonType: "string"
          // minLength retiré
        }
      }
    }
  }
})
```

---

## 🔄 Modification complète du schéma

### Scénario : Refonte majeure

Parfois, vous devez changer significativement la structure.

**Situation** : Passage d'une structure plate à une structure imbriquée

```javascript
// Ancien schéma : structure plate
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "rue", "ville", "codePostal"],
    properties: {
      nom: { bsonType: "string" },
      rue: { bsonType: "string" },
      ville: { bsonType: "string" },
      codePostal: { bsonType: "string" }
    }
  }
}

// Nouveau schéma : structure imbriquée
db.runCommand({
  collMod: "clients",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "adresse"],
      properties: {
        nom: { bsonType: "string" },
        adresse: {  // Nouvelle structure imbriquée
          bsonType: "object",
          required: ["rue", "ville", "codePostal"],
          properties: {
            rue: { bsonType: "string" },
            ville: { bsonType: "string" },
            codePostal: { bsonType: "string" }
          }
        }
      }
    }
  },
  validationLevel: "moderate",  // Important !
  validationAction: "warn"      // Important !
})
```

**Plan de migration** :

1. **Phase 1** : Activer le nouveau schéma en mode `moderate` + `warn`
2. **Phase 2** : L'application écrit dans les deux formats (ancien et nouveau)
3. **Phase 3** : Migration progressive des anciens documents
4. **Phase 4** : L'application ne lit que le nouveau format
5. **Phase 5** : Suppression des anciens champs
6. **Phase 6** : Passage en mode `strict` + `error`

---

## 📋 Stratégies de modification sécurisées

### Stratégie 1 : Modification progressive (recommandée)

Pour des changements importants, procédez par étapes :

```javascript
// Étape 1 : Ajouter le nouveau champ comme facultatif (warn)
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom"],  // "prix" pas encore requis
      properties: {
        nom: { bsonType: "string" },
        prix: { bsonType: "double", minimum: 0 }  // Facultatif
      }
    }
  },
  validationAction: "warn"
})

// Étape 2 : Attendre que les documents soient mis à jour
// (l'application ajoute progressivement le champ prix)

// Étape 3 : Vérifier le taux de conformité
const total = db.produits.countDocuments()
const avecPrix = db.produits.countDocuments({ prix: { $exists: true } })
const taux = (avecPrix / total * 100).toFixed(2)
print(`${taux}% des produits ont un prix`)

// Étape 4 : Quand > 99%, rendre le champ obligatoire
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix"],  // Maintenant requis
      properties: {
        nom: { bsonType: "string" },
        prix: { bsonType: "double", minimum: 0 }
      }
    }
  },
  validationAction: "error"  // Mode strict
})
```

### Stratégie 2 : Test sur un échantillon

Avant d'appliquer à toute la collection, testez sur un sous-ensemble :

```javascript
// 1. Créer une collection de test avec quelques documents
db.produits.aggregate([
  { $sample: { size: 1000 } },  // 1000 documents aléatoires
  { $out: "produits_test" }
])

// 2. Appliquer les nouvelles règles sur la collection de test
db.runCommand({
  collMod: "produits_test",
  validator: {
    $jsonSchema: {
      // Nouvelles règles
    }
  }
})

// 3. Tester les opérations
try {
  db.produits_test.updateOne(
    { _id: ObjectId("...") },
    { $set: { /* modifications */ } }
  )
  print("✅ Test réussi")
} catch (e) {
  print("❌ Test échoué:", e.message)
}

// 4. Si OK, appliquer sur la vraie collection
```

### Stratégie 3 : Modification par version

Pour des changements majeurs, utilisez le versioning :

```javascript
// Ajouter un champ version au schéma
db.runCommand({
  collMod: "documents",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["version"],
      properties: {
        version: {
          enum: [1, 2],  // Accepter v1 et v2
          description: "Version du schéma"
        }
      },
      oneOf: [
        {
          // Schéma version 1
          properties: {
            version: { const: 1 },
            // Ancienne structure
          }
        },
        {
          // Schéma version 2
          properties: {
            version: { const: 2 },
            // Nouvelle structure
          }
        }
      ]
    }
  }
})
```

### Stratégie 4 : Rollback plan

Ayez toujours un plan de retour arrière :

```javascript
// AVANT modification : sauvegarder le schéma actuel
const backupValidator = db.getCollectionInfos({ name: "produits" })[0].options.validator
const backupLevel = db.getCollectionInfos({ name: "produits" })[0].options.validationLevel
const backupAction = db.getCollectionInfos({ name: "produits" })[0].options.validationAction

// Sauvegarder dans une collection de backup
db.schema_backups.insertOne({
  collection: "produits",
  date: new Date(),
  validator: backupValidator,
  validationLevel: backupLevel,
  validationAction: backupAction
})

// Appliquer la modification
db.runCommand({
  collMod: "produits",
  validator: { /* nouvelles règles */ }
})

// Si problème : rollback
function rollback(collectionName) {
  const backup = db.schema_backups.findOne(
    { collection: collectionName },
    { sort: { date: -1 } }  // Plus récent
  )

  if (!backup) {
    print("Aucune sauvegarde trouvée !")
    return
  }

  db.runCommand({
    collMod: collectionName,
    validator: backup.validator,
    validationLevel: backup.validationLevel,
    validationAction: backup.validationAction
  })

  print("Rollback effectué !")
}
```

---

## 🎯 Cas d'usage réels

### Cas 1 : Ajout d'un champ obligatoire sur production

**Contexte** : Application e-commerce avec 500k produits. Nouveau besoin : tous les produits doivent avoir une catégorie.

**Solution** :

```javascript
// Phase 1 : Ajouter "categorie" comme facultatif (semaine 1)
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix"],  // categorie pas encore requis
      properties: {
        nom: { bsonType: "string" },
        prix: { bsonType: "double", minimum: 0 },
        categorie: {
          enum: ["electronique", "vetement", "alimentation", "autre"]
        }
      }
    }
  },
  validationLevel: "moderate",
  validationAction: "warn"
})

// Phase 2 : Script de migration (semaine 2-3)
db.produits.updateMany(
  { categorie: { $exists: false } },
  { $set: { categorie: "autre" } }  // Valeur par défaut
)

// Phase 3 : Vérification (semaine 4)
const sansCategorie = db.produits.countDocuments({ categorie: { $exists: false } })
print(`Produits sans catégorie: ${sansCategorie}`)

// Phase 4 : Rendre obligatoire (semaine 5)
if (sansCategorie === 0) {
  db.runCommand({
    collMod: "produits",
    validator: {
      $jsonSchema: {
        bsonType: "object",
        required: ["nom", "prix", "categorie"],  // Maintenant requis !
        properties: {
          nom: { bsonType: "string" },
          prix: { bsonType: "double", minimum: 0 },
          categorie: {
            enum: ["electronique", "vetement", "alimentation", "autre"]
          }
        }
      }
    },
    validationLevel: "strict",
    validationAction: "error"
  })
  print("✅ Catégorie maintenant obligatoire")
}
```

### Cas 2 : Correction d'une regex trop stricte

**Contexte** : La regex du téléphone rejette des numéros valides.

**Problème** :
```javascript
// Regex actuelle : trop stricte
telephone: {
  bsonType: "string",
  pattern: "^06[0-9]{8}$"  // Uniquement 06... (mobiles)
}
// Rejette les fixes (01, 02, etc.)
```

**Solution immédiate** :

```javascript
// Passer en mode warn pendant la correction
db.runCommand({
  collMod: "contacts",
  validationAction: "warn"  // Accepter temporairement tout
})

// Corriger la regex
db.runCommand({
  collMod: "contacts",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        telephone: {
          bsonType: "string",
          pattern: "^0[1-9][0-9]{8}$"  // Corrigé : tous les numéros FR
        }
      }
    }
  },
  validationAction: "error"  // Réactiver le blocage
})
```

### Cas 3 : Ajout de validation à une collection existante

**Contexte** : Collection créée il y a 2 ans sans validation. Maintenant besoin de valider.

**Solution** :

```javascript
// Étape 1 : Analyser les données existantes
const echantillon = db.utilisateurs.aggregate([
  { $sample: { size: 1000 } },
  { $project: {
    hasNom: { $cond: [{ $ne: ["$nom", null] }, 1, 0] },
    hasEmail: { $cond: [{ $ne: ["$email", null] }, 1, 0] },
    emailFormat: { $regexMatch: { input: "$email", regex: "^.+@.+\\..+$" } }
  }},
  { $group: {
    _id: null,
    totalNom: { $sum: "$hasNom" },
    totalEmail: { $sum: "$hasEmail" },
    emailsValides: { $sum: { $cond: ["$emailFormat", 1, 0] } },
    count: { $sum: 1 }
  }}
]).toArray()[0]

print("Analyse de l'échantillon :")
print(`Avec nom: ${(echantillon.totalNom / echantillon.count * 100).toFixed(1)}%`)
print(`Avec email: ${(echantillon.totalEmail / echantillon.count * 100).toFixed(1)}%`)
print(`Emails valides: ${(echantillon.emailsValides / echantillon.totalEmail * 100).toFixed(1)}%`)

// Étape 2 : Créer schéma basé sur l'analyse
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom"],  // 100% ont un nom
      properties: {
        nom: { bsonType: "string", minLength: 1 },
        email: {  // Seulement 80% ont email -> facultatif
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        }
      }
    }
  },
  validationLevel: "moderate",  // Important !
  validationAction: "warn"       // Important !
})

// Étape 3 : Migration progressive...
```

### Cas 4 : Changement de type de données

**Contexte** : Le champ `quantite` était stocké en string, doit devenir int.

**Solution** :

```javascript
// Phase 1 : Accepter les deux types temporairement
db.runCommand({
  collMod: "stock",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        quantite: {
          bsonType: ["string", "int"]  // Les deux acceptés
        }
      }
    }
  },
  validationLevel: "moderate"
})

// Phase 2 : Migration des données
db.stock.updateMany(
  { quantite: { $type: "string" } },
  [{ $set: { quantite: { $toInt: "$quantite" } } }]
)

// Phase 3 : Vérification
const encore String = db.stock.countDocuments({ quantite: { $type: "string" } })
print(`Documents avec quantite en string: ${encoreString}`)

// Phase 4 : Forcer le type int uniquement
if (encoreString === 0) {
  db.runCommand({
    collMod: "stock",
    validator: {
      $jsonSchema: {
        bsonType: "object",
        properties: {
          quantite: {
            bsonType: "int"  // Uniquement int maintenant
          }
        }
      }
    },
    validationLevel: "strict"
  })
}
```

---

## 💡 Bonnes pratiques

### 1. Toujours sauvegarder avant modification

```javascript
// Script de sauvegarde automatique
function backupValidation(collectionName) {
  const info = db.getCollectionInfos({ name: collectionName })[0]

  if (!info) {
    print(`Collection "${collectionName}" n'existe pas`)
    return false
  }

  const backup = {
    collection: collectionName,
    date: new Date(),
    validator: info.options.validator,
    validationLevel: info.options.validationLevel,
    validationAction: info.options.validationAction
  }

  db.validation_backups.insertOne(backup)
  print(`✅ Sauvegarde créée pour ${collectionName}`)
  return true
}

// Utilisation avant chaque modification
backupValidation("produits")
```

### 2. Tester sur environnement de staging

Ne jamais appliquer directement en production :

```javascript
// ❌ DANGEREUX
db.runCommand({
  collMod: "produits_production",
  validator: { /* nouvelles règles non testées */ }
})

// ✅ BON
// 1. Tester en développement
// 2. Tester en staging
// 3. Déployer en production avec mode warn
// 4. Observer
// 5. Passer en mode error
```

### 3. Documenter les changements

```javascript
// Ajouter un commentaire dans le schéma
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      title: "Schéma produits v2.1",
      description: "Ajout du champ categorie obligatoire - 2025-01-15 - Ticket PROJ-456",
      required: ["nom", "prix", "categorie"],
      properties: {
        // ...
      }
    }
  }
})

// Maintenir un changelog
db.schema_changelog.insertOne({
  collection: "produits",
  date: new Date(),
  version: "2.1",
  changes: [
    "Ajout champ 'categorie' obligatoire",
    "Modification pattern email pour accepter nouveaux TLD"
  ],
  author: "jean.dupont",
  ticket: "PROJ-456"
})
```

### 4. Utiliser le versioning pour changements majeurs

```javascript
// Méthode avec champ de version
{
  version: 2,  // Version du schéma
  nom: "Produit",
  // ... autres champs selon version
}

// Le validator gère plusieurs versions
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      oneOf: [
        { properties: { version: { const: 1 } /* schéma v1 */ } },
        { properties: { version: { const: 2 } /* schéma v2 */ } }
      ]
    }
  }
})
```

### 5. Monitoring post-modification

```javascript
// Script de monitoring après modification
function monitorValidation(collectionName, durationMinutes) {
  print(`Monitoring de ${collectionName} pendant ${durationMinutes} minutes...`)

  const startTime = new Date()
  const endTime = new Date(startTime.getTime() + durationMinutes * 60000)

  // Vérifier périodiquement
  while (new Date() < endTime) {
    const errors = db.adminCommand({ getLog: "global" }).log.filter(
      line => line.includes("Document failed validation") &&
              line.includes(collectionName)
    ).length

    print(`[${new Date().toLocaleTimeString()}] Erreurs de validation: ${errors}`)

    sleep(60000)  // Attendre 1 minute
  }

  print("Monitoring terminé")
}

// Utilisation après modification
monitorValidation("produits", 30)  // Monitor 30 minutes
```

---

## ⚠️ Pièges à éviter

### 1. Oublier de copier toutes les règles existantes

```javascript
// ❌ ERREUR : Oubli des propriétés existantes
// Règles actuelles
{
  required: ["nom", "prix", "stock"],
  properties: {
    nom: { bsonType: "string" },
    prix: { bsonType: "double" },
    stock: { bsonType: "int" }
  }
}

// Modification qui OUBLIE "prix" et "stock"
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      required: ["nom"],  // prix et stock disparus !
      properties: {
        nom: { bsonType: "string" }
        // prix et stock pas inclus = plus de validation !
      }
    }
  }
})

// ✅ CORRECT : Inclure TOUTES les règles
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      required: ["nom", "prix", "stock"],
      properties: {
        nom: { bsonType: "string" },
        prix: { bsonType: "double" },
        stock: { bsonType: "int" }
      }
    }
  }
})
```

### 2. Modifier en mode strict sans préparation

```javascript
// ❌ DANGER : Rendre champ obligatoire en strict directement
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      required: ["email"]  // Email maintenant requis
    }
  },
  validationLevel: "strict"  // Tous les documents doivent avoir email
})
// Peut bloquer la modification de milliers de documents existants !

// ✅ BON : Progression
// 1. Mode moderate + warn
// 2. Migration des données
// 3. Mode strict + error
```

### 3. Ne pas tester les regex

```javascript
// ❌ Regex non testée
pattern: "^[0-9]{5}$"  // Est-ce que ça marche ?

// ✅ Tester la regex d'abord
const testValues = ["75001", "1234", "ABCDE", "750011"]
const regex = /^[0-9]{5}$/

testValues.forEach(val => {
  print(`"${val}" -> ${regex.test(val) ? "✅" : "❌"}`)
})
```

### 4. Modifier en production aux heures de pointe

```javascript
// ❌ Modification vendredi 18h
// ❌ Modification pendant les soldes
// ❌ Modification sans planification

// ✅ Planifier les modifications
// - Heures creuses (nuit, weekend)
// - Avec présence de l'équipe
// - Avec plan de rollback
// - Communication préalable
```

### 5. Ignorer les avertissements en mode warn

```javascript
// ❌ Activer warn et ne jamais vérifier
db.runCommand({
  collMod: "produits",
  validationAction: "warn"
})
// 3 mois plus tard... toujours des violations !

// ✅ Surveiller et agir
// - Analyser les logs régulièrement
// - Corriger les problèmes détectés
// - Passer en error une fois propre
```

---

## 🎓 Résumé

### Commande principale

```javascript
db.runCommand({
  collMod: "nomCollection",
  validator: { $jsonSchema: { /* schéma COMPLET */ } },
  validationLevel: "strict" | "moderate",
  validationAction: "error" | "warn"
})
```

### Checklist de modification

✅ **Avant modification** :
- [ ] Sauvegarder le schéma actuel
- [ ] Analyser les données existantes
- [ ] Tester sur environnement de staging
- [ ] Planifier la migration si nécessaire
- [ ] Communiquer avec l'équipe

✅ **Pendant modification** :
- [ ] Inclure TOUTES les règles existantes
- [ ] Commencer en mode warn ou moderate
- [ ] Documenter les changements
- [ ] Avoir un plan de rollback

✅ **Après modification** :
- [ ] Monitoring des logs
- [ ] Vérifier les taux d'erreur
- [ ] Corriger les problèmes détectés
- [ ] Passer en mode strict progressivement

### Ordre de sécurité (du plus sûr au moins sûr)

1. `moderate` + `warn` → Observation sans risque
2. `moderate` + `error` → Validation des nouveaux seulement
3. `strict` + `warn` → Validation de tout en observation
4. `strict` + `error` → Validation stricte complète

### Points clés

- ✅ `collMod` **remplace complètement** le schéma (pas de merge)
- ✅ Toujours **sauvegarder** avant modification
- ✅ **Tester** sur staging avant production
- ✅ Procéder par **étapes progressives**
- ✅ **Monitorer** après chaque changement
- ✅ **Documenter** toutes les modifications

---

## 📚 Dans la prochaine section

Dans la section suivante (7.7), nous verrons comment valider les **types de données** de manière détaillée avec des exemples pratiques pour chaque type BSON.

---


⏭️ [Validation des types de données](/07-validation-des-schemas/07-validation-types-donnees.md)
