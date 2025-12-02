🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.10 Bonnes pratiques de validation

## 📚 Vue d'ensemble

Cette section rassemble toutes les **bonnes pratiques** pour créer et maintenir des schémas de validation MongoDB robustes, performants et maintenables. Ces recommandations sont issues de l'expérience de nombreux projets en production.

---

## 🎯 Principes généraux

### 1. Commencer simple, complexifier progressivement

**Principe** : Ne créez pas un schéma complexe dès le départ. Commencez avec l'essentiel et ajoutez des règles au fur et à mesure.

```javascript
// ❌ ÉVITER : Tout dès le début
db.createCollection("utilisateurs", {
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["nom", "prenom", "email", "telephone", "adresse",
                     "ville", "code_postal", "pays", "date_naissance",
                     "profession", "entreprise", "siret"],
          properties: {
            // 20+ propriétés avec validations complexes
          }
        }
      },
      {
        $expr: {
          // 15 règles métier complexes
        }
      }
    ]
  }
})

// ✅ BON : Étape 1 - Minimum vital
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        }
      }
    }
  }
})

// ✅ BON : Étape 2 - Ajout progressif (quelques semaines plus tard)
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "nom"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        },
        nom: {
          bsonType: "string",
          minLength: 2,
          maxLength: 50
        }
      }
    }
  }
})
```

### 2. Privilégier la clarté à la concision

**Principe** : Un schéma lisible est plus important qu'un schéma court.

```javascript
// ❌ ÉVITER : Trop concis, difficile à comprendre
{
  $jsonSchema: {
    required: ["n", "p", "q"],
    properties: {
      n: { bsonType: "string", minLength: 2 },
      p: { bsonType: "decimal", minimum: 0 },
      q: { bsonType: "int", minimum: 1 }
    }
  }
}

// ✅ BON : Clair et explicite
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "prix", "quantite"],
    properties: {
      nom: {
        bsonType: "string",
        minLength: 2,
        maxLength: 100,
        description: "Nom du produit (2-100 caractères)"
      },
      prix: {
        bsonType: "decimal",
        minimum: 0,
        description: "Prix unitaire en euros TTC"
      },
      quantite: {
        bsonType: "int",
        minimum: 1,
        description: "Quantité en stock"
      }
    }
  }
}
```

### 3. Documenter systématiquement

**Principe** : Chaque règle doit avoir une `description` qui explique son but.

```javascript
// ✅ BON : Chaque champ documenté
{
  $jsonSchema: {
    title: "Schéma Produit v2.0",
    description: "Validation des produits e-commerce. Dernière mise à jour : 2025-01-15",
    bsonType: "object",
    required: ["nom", "prix"],
    properties: {
      nom: {
        bsonType: "string",
        minLength: 3,
        maxLength: 200,
        description: "Nom commercial du produit (3-200 caractères). Doit être unique."
      },
      prix: {
        bsonType: "decimal",
        minimum: 0,
        exclusiveMinimum: 0,
        description: "Prix de vente TTC en euros. Doit être strictement positif."
      },
      stock: {
        bsonType: "int",
        minimum: 0,
        description: "Quantité disponible en stock. 0 = rupture de stock."
      }
    }
  }
}
```

---

## 📐 Conception du schéma

### 1. Séparer structure et logique métier

**Principe** : Utilisez `$jsonSchema` pour la structure et `$expr` pour les règles métier.

```javascript
// ✅ BON : Séparation claire
{
  validator: {
    $and: [
      // Structure et types
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["date_debut", "date_fin", "prix"],
          properties: {
            date_debut: { bsonType: "date" },
            date_fin: { bsonType: "date" },
            prix: { bsonType: "decimal", minimum: 0 }
          }
        }
      },
      // Règles métier
      {
        $expr: {
          $and: [
            { $lt: ["$date_debut", "$date_fin"] },
            { $gte: ["$date_debut", "$$NOW"] }
          ]
        }
      }
    ]
  }
}
```

### 2. Éviter la sur-validation

**Principe** : Ne validez que ce qui est nécessaire. Laissez de la flexibilité.

```javascript
// ❌ TROP STRICT : Bloque l'évolution
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email"],
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" }
    },
    additionalProperties: false  // Interdit tout nouveau champ !
  }
}

// ✅ BON : Flexible
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email"],
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" }
    }
    // additionalProperties: true par défaut
    // Permet d'ajouter de nouveaux champs sans casser la validation
  }
}
```

### 3. Valider les champs critiques en priorité

**Principe** : Concentrez-vous sur les champs essentiels au fonctionnement.

```javascript
// ✅ BON : Priorités claires
{
  $jsonSchema: {
    bsonType: "object",
    // Critiques : obligatoires avec validation stricte
    required: ["email", "mot_de_passe", "statut"],
    properties: {
      email: {
        bsonType: "string",
        pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
        description: "CRITIQUE : Email pour connexion"
      },
      mot_de_passe: {
        bsonType: "string",
        minLength: 8,
        description: "CRITIQUE : Hash du mot de passe"
      },
      statut: {
        enum: ["actif", "inactif", "suspendu"],
        description: "CRITIQUE : Statut du compte"
      },
      // Secondaires : facultatifs, validation légère
      telephone: {
        bsonType: "string",
        description: "SECONDAIRE : Téléphone de contact"
      },
      preferences: {
        bsonType: "object",
        description: "SECONDAIRE : Préférences utilisateur"
      }
    }
  }
}
```

### 4. Utiliser des valeurs par défaut côté application

**Principe** : MongoDB ne supporte pas les valeurs par défaut dans les schémas. Gérez-les dans l'application.

```javascript
// ✅ BON : Valeurs par défaut dans l'application
function creerUtilisateur(email, nom) {
  return db.utilisateurs.insertOne({
    email: email,
    nom: nom,
    date_creation: new Date(),        // Défaut : maintenant
    statut: "actif",                  // Défaut : actif
    newsletter: true,                 // Défaut : inscrit
    role: "utilisateur"               // Défaut : utilisateur
  })
}
```

---

## 📊 Organisation et maintenance

### 1. Versioner vos schémas

**Principe** : Gardez une trace des évolutions de vos schémas.

```javascript
// Collection pour historiser les schémas
db.schema_versions.insertOne({
  collection: "utilisateurs",
  version: "2.0",
  date: new Date("2025-01-15"),
  auteur: "jean.dupont",
  changements: [
    "Ajout champ 'telephone' obligatoire",
    "Modification pattern email pour accepter nouveaux TLD"
  ],
  validator: {
    $jsonSchema: {
      title: "Schéma Utilisateurs v2.0",
      // ... schéma complet
    }
  }
})

// Ou utiliser un champ version dans le schéma lui-même
{
  $jsonSchema: {
    title: "Schéma Produits v3.1",
    description: "Version 3.1 - 2025-01-15 - Ajout validation stock négatif",
    // ...
  }
}
```

### 2. Externaliser les schémas complexes

**Principe** : Pour les schémas volumineux, stockez-les dans des fichiers séparés.

```javascript
// schemas/utilisateurs.js
module.exports = {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "nom"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        nom: {
          bsonType: "string",
          minLength: 2
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
}

// Application
const utilisateursSchema = require('./schemas/utilisateurs')
db.createCollection("utilisateurs", utilisateursSchema)
```

### 3. Créer une bibliothèque de patterns réutilisables

**Principe** : Définissez des patterns communs pour la cohérence.

```javascript
// patterns/common.js
const CommonPatterns = {
  email: {
    bsonType: "string",
    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
    description: "Adresse email valide"
  },

  telephoneFR: {
    bsonType: "string",
    pattern: "^(\\+33|0)[1-9][0-9]{8}$",
    description: "Numéro de téléphone français"
  },

  codePostalFR: {
    bsonType: "string",
    pattern: "^[0-9]{5}$",
    description: "Code postal français (5 chiffres)"
  },

  url: {
    bsonType: "string",
    pattern: "^https?:\\/\\/(www\\.)?[-a-zA-Z0-9@:%._\\+~#=]{1,256}\\.[a-zA-Z0-9()]{1,6}\\b",
    description: "URL valide (HTTP ou HTTPS)"
  },

  objectId: {
    bsonType: "objectId",
    description: "Identifiant MongoDB"
  },

  prixEuros: {
    bsonType: "decimal",
    minimum: 0,
    description: "Prix en euros (positif)"
  }
}

module.exports = CommonPatterns

// Utilisation
const CommonPatterns = require('./patterns/common')

db.createCollection("clients", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "telephone"],
      properties: {
        email: CommonPatterns.email,
        telephone: CommonPatterns.telephoneFR,
        code_postal: CommonPatterns.codePostalFR
      }
    }
  }
})
```

### 4. Automatiser la validation des schémas

**Principe** : Créez des scripts de test pour vos schémas.

```javascript
// tests/schema-validator.js
function testSchema(collectionName, validCases, invalidCases) {
  console.log(`\nTest du schéma : ${collectionName}`)

  // Tester les cas valides
  validCases.forEach((doc, index) => {
    try {
      db[collectionName].insertOne(doc)
      db[collectionName].deleteOne({ _id: doc._id })
      console.log(`✅ Cas valide ${index + 1} : OK`)
    } catch (e) {
      console.error(`❌ Cas valide ${index + 1} : ÉCHEC (devrait être accepté)`)
    }
  })

  // Tester les cas invalides
  invalidCases.forEach((doc, index) => {
    try {
      db[collectionName].insertOne(doc)
      db[collectionName].deleteOne({ _id: doc._id })
      console.error(`❌ Cas invalide ${index + 1} : ÉCHEC (devrait être rejeté)`)
    } catch (e) {
      console.log(`✅ Cas invalide ${index + 1} : OK (correctement rejeté)`)
    }
  })
}

// Utilisation
testSchema("utilisateurs",
  // Cas valides
  [
    { email: "test@example.com", nom: "Dupont" },
    { email: "user@domain.fr", nom: "Martin", telephone: "0612345678" }
  ],
  // Cas invalides
  [
    { email: "invalid-email", nom: "Test" },
    { email: "test@example.com" }, // nom manquant
    { email: "test@example.com", nom: "A" } // nom trop court
  ]
)
```

---

## ⚡ Performance et optimisation

### 1. Préférer $jsonSchema à $expr quand possible

**Principe** : `$jsonSchema` est plus performant que `$expr`.

```javascript
// ❌ MOINS PERFORMANT : Tout avec $expr
{
  validator: {
    $expr: {
      $and: [
        { $eq: [{ $type: "$nom" }, "string"] },
        { $gte: [{ $strLenCP: "$nom" }, 2] }
      ]
    }
  }
}

// ✅ PLUS PERFORMANT : $jsonSchema quand possible
{
  validator: {
    $jsonSchema: {
      properties: {
        nom: {
          bsonType: "string",
          minLength: 2
        }
      }
    }
  }
}
```

### 2. Limiter la complexité des validations

**Principe** : Les validations trop complexes ralentissent les insertions/modifications.

```javascript
// ❌ TROP COMPLEXE : Impact performance significatif
{
  validator: {
    $expr: {
      $and: [
        // 20+ conditions imbriquées
        // Calculs complexes sur tableaux
        // Multiples $reduce, $map, etc.
      ]
    }
  }
}

// ✅ MIEUX : Valider l'essentiel, le reste côté application
{
  validator: {
    $and: [
      { $jsonSchema: { /* structure de base */ } },
      {
        $expr: {
          // Seulement les règles critiques
          $lt: ["$date_debut", "$date_fin"]
        }
      }
    ]
  }
}
```

### 3. Tester l'impact sur les performances

**Principe** : Mesurez l'impact de vos validations sur les opérations.

```javascript
// Script de benchmark
function benchmarkValidation(collectionName, documents) {
  const start = Date.now()

  documents.forEach(doc => {
    try {
      db[collectionName].insertOne(doc)
    } catch (e) {
      // Ignorer les erreurs de validation pour le benchmark
    }
  })

  const end = Date.now()
  const duration = end - start
  const avgTime = duration / documents.length

  console.log(`Collection: ${collectionName}`)
  console.log(`Documents: ${documents.length}`)
  console.log(`Durée totale: ${duration}ms`)
  console.log(`Temps moyen par document: ${avgTime.toFixed(2)}ms`)

  // Nettoyer
  db[collectionName].deleteMany({})
}

// Test avec 1000 documents
const testDocs = Array(1000).fill().map((_, i) => ({
  nom: `Produit ${i}`,
  prix: Math.random() * 100
}))

benchmarkValidation("produits", testDocs)
```

---

## 🔒 Sécurité et qualité des données

### 1. Valider les formats sensibles

**Principe** : Soyez particulièrement strict sur les données sensibles.

```javascript
{
  $jsonSchema: {
    properties: {
      // Email : validation stricte
      email: {
        bsonType: "string",
        pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
        description: "Email valide requis pour connexion"
      },

      // Mot de passe : hash uniquement
      mot_de_passe_hash: {
        bsonType: "string",
        minLength: 60,
        maxLength: 60,
        pattern: "^\\$2[aby]\\$",  // bcrypt hash
        description: "Hash bcrypt du mot de passe"
      },

      // Numéro de carte : ne jamais stocker en clair !
      derniers_chiffres_carte: {
        bsonType: "string",
        pattern: "^[0-9]{4}$",
        description: "4 derniers chiffres de la carte uniquement"
      },

      // IBAN : format européen
      iban: {
        bsonType: "string",
        pattern: "^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$",
        description: "IBAN au format standard"
      }
    }
  }
}
```

### 2. Empêcher les injections dans les champs texte

**Principe** : Limitez la longueur et les caractères acceptés.

```javascript
{
  $jsonSchema: {
    properties: {
      nom: {
        bsonType: "string",
        minLength: 2,
        maxLength: 100,  // Limite raisonnable
        description: "Nom (2-100 caractères)"
      },

      commentaire: {
        bsonType: "string",
        maxLength: 1000,  // Empêche les abus
        description: "Commentaire (max 1000 caractères)"
      },

      url_redirect: {
        bsonType: "string",
        pattern: "^https:\\/\\/mondomaine\\.com\\/.+$",  // Domaine restreint
        description: "URL de redirection (uniquement notre domaine)"
      }
    }
  }
}
```

### 3. Valider les relations entre collections

**Principe** : Assurez-vous que les références sont valides.

```javascript
// Note : MongoDB ne peut pas valider les références croisées automatiquement
// Faites-le côté application

// Exemple : Vérifier que l'auteur existe avant d'insérer un article
async function creerArticle(titre, auteur_id) {
  // Vérifier que l'auteur existe
  const auteur = await db.utilisateurs.findOne({ _id: auteur_id })
  if (!auteur) {
    throw new Error("Auteur inexistant")
  }

  // Valider avec le schéma MongoDB
  return db.articles.insertOne({
    titre: titre,
    auteur_id: auteur_id,
    date_creation: new Date()
  })
}
```

---

## 🚀 Déploiement et migration

### 1. Déployer progressivement

**Principe** : Utilisez une stratégie de déploiement en plusieurs phases.

```javascript
// Phase 1 : Mode warn (2 semaines)
db.runCommand({
  collMod: "produits",
  validator: { /* nouvelles règles */ },
  validationLevel: "moderate",
  validationAction: "warn"
})

// Phase 2 : Mode moderate + error (2 semaines)
db.runCommand({
  collMod: "produits",
  validationLevel: "moderate",
  validationAction: "error"
})

// Phase 3 : Mode strict (après vérification)
db.runCommand({
  collMod: "produits",
  validationLevel: "strict",
  validationAction: "error"
})
```

### 2. Communiquer les changements

**Principe** : Informez toute l'équipe des modifications de validation.

```javascript
// Changelog
db.schema_changelog.insertOne({
  collection: "utilisateurs",
  version: "2.1",
  date: new Date("2025-01-15"),
  type: "breaking_change",
  description: "Le champ 'telephone' devient obligatoire",
  impact: "Toutes les insertions sans telephone seront rejetées",
  migration: "Exécuter le script migrate-telephone.js avant activation",
  rollback: "Exécuter rollback-telephone.js",
  auteur: "jean.dupont",
  ticket: "PROJ-456",
  notification_envoyee: ["dev-team@example.com", "ops-team@example.com"]
})
```

### 3. Prévoir un plan de rollback

**Principe** : Sauvegardez toujours l'ancien schéma avant modification.

```javascript
// Script de rollback automatique
function applySchemaWithRollback(collectionName, newValidator) {
  // 1. Sauvegarder le schéma actuel
  const currentConfig = db.getCollectionInfos({ name: collectionName })[0]
  const backup = {
    collection: collectionName,
    date: new Date(),
    validator: currentConfig.options.validator,
    validationLevel: currentConfig.options.validationLevel,
    validationAction: currentConfig.options.validationAction
  }
  db.schema_backups.insertOne(backup)

  // 2. Appliquer le nouveau schéma
  try {
    db.runCommand({
      collMod: collectionName,
      validator: newValidator.validator,
      validationLevel: newValidator.validationLevel,
      validationAction: newValidator.validationAction
    })
    console.log(`✅ Nouveau schéma appliqué sur ${collectionName}`)
    return { success: true, backupId: backup._id }
  } catch (e) {
    console.error(`❌ Erreur lors de l'application du schéma : ${e.message}`)
    return { success: false, error: e.message }
  }
}

// Fonction de rollback
function rollbackSchema(backupId) {
  const backup = db.schema_backups.findOne({ _id: backupId })
  if (!backup) {
    throw new Error("Backup introuvable")
  }

  db.runCommand({
    collMod: backup.collection,
    validator: backup.validator,
    validationLevel: backup.validationLevel,
    validationAction: backup.validationAction
  })

  console.log(`✅ Rollback effectué sur ${backup.collection}`)
}
```

### 4. Valider sur environnement de test

**Principe** : Ne jamais appliquer directement en production.

```javascript
// ❌ DANGEREUX
// Appliquer directement en production
db.runCommand({
  collMod: "utilisateurs_production",
  validator: { /* nouveau schéma non testé */ }
})

// ✅ BON : Processus complet
// 1. Développement local
// 2. Tests unitaires
// 3. Environnement de staging
// 4. Tests d'intégration
// 5. Validation métier
// 6. Production avec mode warn
// 7. Production avec mode error
```

---

## 📋 Checklist de validation complète

### Avant la création

- [ ] Identifier les champs vraiment essentiels
- [ ] Définir les types de données appropriés
- [ ] Lister les contraintes métier critiques
- [ ] Consulter les parties prenantes
- [ ] Documenter les choix de conception

### Pendant la création

- [ ] Utiliser `$jsonSchema` pour la structure
- [ ] Ajouter `$expr` pour les règles métier complexes
- [ ] Documenter chaque champ avec `description`
- [ ] Versionner le schéma (`title` avec version)
- [ ] Externaliser dans un fichier si complexe
- [ ] Créer des patterns réutilisables

### Validation du schéma

- [ ] Tester avec des cas valides
- [ ] Tester avec des cas invalides
- [ ] Tester les cas limites (null, 0, dates extrêmes)
- [ ] Vérifier les performances
- [ ] Faire une revue de code
- [ ] Valider avec l'équipe métier

### Déploiement

- [ ] Sauvegarder le schéma actuel
- [ ] Commencer en mode `warn`
- [ ] Surveiller les logs
- [ ] Corriger les problèmes détectés
- [ ] Passer en mode `error` progressivement
- [ ] Communiquer à l'équipe
- [ ] Documenter dans le changelog

### Après déploiement

- [ ] Monitorer les erreurs de validation
- [ ] Analyser les impacts performance
- [ ] Collecter les retours utilisateurs
- [ ] Ajuster si nécessaire
- [ ] Planifier les évolutions futures

---

## 🎯 Anti-patterns à éviter

### 1. Le schéma "tout ou rien"

```javascript
// ❌ Anti-pattern : Validation extrême
{
  $jsonSchema: {
    required: [
      "nom", "prenom", "date_naissance", "lieu_naissance",
      "nationalite", "adresse_complete", "telephone_fixe",
      "telephone_mobile", "email_principal", "email_secondaire",
      "profession", "employeur", "salaire_annuel"
      // ... 30 champs obligatoires
    ]
  }
}
// Résultat : Personne ne peut créer de compte !
```

### 2. La regex incompréhensible

```javascript
// ❌ Anti-pattern : Regex sans explication
{
  pattern: "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$"
}

// ✅ Meilleur : Regex documentée
{
  pattern: "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
  description: `Mot de passe fort requis :
    - Au moins 8 caractères
    - Au moins 1 majuscule
    - Au moins 1 minuscule
    - Au moins 1 chiffre
    - Au moins 1 caractère spécial (@$!%*?&)`
}
```

### 3. La validation dupliquée

```javascript
// ❌ Anti-pattern : Logique répétée partout
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      properties: {
        email: {
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        }
      }
    }
  }
})

db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      properties: {
        email: {
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"  // Dupliqué !
        }
      }
    }
  }
})

// ✅ Meilleur : Patterns centralisés
const CommonPatterns = require('./patterns/common')

db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      properties: {
        email: CommonPatterns.email
      }
    }
  }
})
```

### 4. Le schéma non versionné

```javascript
// ❌ Anti-pattern : Pas de traçabilité
{
  $jsonSchema: {
    // ... schéma
  }
}

// ✅ Meilleur : Versioning clair
{
  $jsonSchema: {
    title: "Schéma Produits v3.2",
    description: "Version 3.2 - 2025-01-15 - Ajout validation catégories",
    // ... schéma
  }
}
```

### 5. L'oubli de la documentation

```javascript
// ❌ Anti-pattern : Aucune documentation
{
  $jsonSchema: {
    required: ["x", "y", "z"],
    properties: {
      x: { bsonType: "int", minimum: 0 },
      y: { bsonType: "string", pattern: "^[A-Z]{3}$" },
      z: { bsonType: "array" }
    }
  }
}
// Dans 6 mois, personne ne sait ce que représentent x, y, z !
```

---

## 📚 Ressources et outils

### Documentation officielle

- MongoDB Manual - Schema Validation
- JSON Schema Documentation
- BSON Specification

### Outils de validation

```javascript
// Générateur de schéma depuis des documents existants
function generateSchemaFromSample(collectionName, sampleSize = 100) {
  const sample = db[collectionName].aggregate([
    { $sample: { size: sampleSize } }
  ]).toArray()

  const schema = {
    bsonType: "object",
    properties: {}
  }

  // Analyser les types de champs
  const fieldTypes = {}
  sample.forEach(doc => {
    Object.keys(doc).forEach(field => {
      const type = typeof doc[field]
      if (!fieldTypes[field]) {
        fieldTypes[field] = new Set()
      }
      fieldTypes[field].add(type)
    })
  })

  // Générer les propriétés
  Object.keys(fieldTypes).forEach(field => {
    const types = Array.from(fieldTypes[field])
    schema.properties[field] = {
      bsonType: types.length === 1 ? types[0] : types,
      description: `Type(s) détecté(s): ${types.join(", ")}`
    }
  })

  return schema
}

// Utilisation
const schema = generateSchemaFromSample("utilisateurs", 100)
printjson(schema)
```

### Validation côté application

```javascript
// Exemple avec Joi (Node.js)
const Joi = require('joi')

const utilisateurSchema = Joi.object({
  email: Joi.string().email().required(),
  nom: Joi.string().min(2).max(50).required(),
  age: Joi.number().integer().min(18).max(120),
  telephone: Joi.string().pattern(/^0[1-9][0-9]{8}$/)
})

// Valider avant insertion
const { error, value } = utilisateurSchema.validate(data)
if (error) {
  throw new Error(`Validation error: ${error.message}`)
}

db.utilisateurs.insertOne(value)
```

---

## 🎓 Résumé final

### Les 10 commandements de la validation MongoDB

1. **Tu commenceras simple** et complexifieras progressivement
2. **Tu documenteras** chaque règle et chaque changement
3. **Tu sépareras** structure ($jsonSchema) et logique métier ($expr)
4. **Tu versionneras** tous tes schémas
5. **Tu testeras** avant de déployer en production
6. **Tu déploieras** progressivement (warn → moderate → strict)
7. **Tu monitoreras** l'impact de tes validations
8. **Tu communiqueras** les changements à ton équipe
9. **Tu prévoiras** un plan de rollback
10. **Tu n'oublieras jamais** que la flexibilité est une force de MongoDB

### Hiérarchie des priorités

```
1. Champs critiques obligatoires avec validation stricte
   ↓
2. Champs importants avec validation modérée
   ↓
3. Champs secondaires avec validation légère
   ↓
4. Champs optionnels sans validation
   ↓
5. Flexibilité pour l'évolution future
```

### Quand utiliser quoi

| Besoin | Outil | Priorité |
|--------|-------|----------|
| Types de données | `$jsonSchema` | Haute |
| Formats (regex) | `$jsonSchema` | Haute |
| Champs obligatoires | `$jsonSchema` | Haute |
| Comparaison champs | `$expr` | Moyenne |
| Calculs | `$expr` | Moyenne |
| Règles temporelles | `$expr` + `$$NOW` | Moyenne |
| Logique complexe | Application | Basse |
| Références croisées | Application | Basse |

### Points clés à retenir

- ✅ La validation est un **équilibre** entre rigidité et flexibilité
- ✅ **Documenter** est aussi important que coder
- ✅ **Tester** sur tous les environnements avant production
- ✅ **Déployer progressivement** avec monitoring
- ✅ **Prévoir l'évolution** : MongoDB est flexible, vos schémas doivent l'être aussi
- ✅ **Communiquer** : La validation impacte toute l'équipe
- ✅ La validation **côté base** complète mais ne remplace pas la validation **côté application**

---

## 🎉 Conclusion

La validation de schéma dans MongoDB est un outil puissant qui vous permet de garantir la qualité de vos données tout en conservant la flexibilité qui fait la force de MongoDB.

En suivant ces bonnes pratiques, vous créerez des schémas :
- ✅ **Robustes** : Qui protègent l'intégrité de vos données
- ✅ **Maintenables** : Faciles à comprendre et à faire évoluer
- ✅ **Performants** : Sans impact négatif significatif
- ✅ **Évolutifs** : Qui s'adaptent aux besoins futurs

N'oubliez pas : un bon schéma de validation est celui qui sert votre application, pas celui qui la contraint.

---

⏭️ [Transactions](/08-transactions/README.md)
