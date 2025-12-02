🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5 Actions de validation (error, warn)

## 📚 Vue d'ensemble

Les **actions de validation** (`validationAction`) déterminent **ce qui se passe** quand un document ne respecte pas les règles de validation. C'est comme choisir la conséquence d'une infraction : soit vous bloquez l'action, soit vous la laissez passer en enregistrant un avertissement.

MongoDB propose deux actions de validation :
- **`error`** : Refuse l'opération et retourne une erreur (par défaut)
- **`warn`** : Accepte l'opération mais enregistre un avertissement dans les logs

---

## 🤔 Pourquoi deux actions ?

### Le problème du tout ou rien

Imaginez cette situation :

1. Vous voulez **tester** de nouvelles règles de validation
2. Mais vous ne voulez pas **casser** votre application en production
3. Vous voulez **observer** combien de documents seraient rejetés
4. Sans **bloquer** les opérations normales

**Dilemme** :
- ❓ En mode `error` : vous bloquez peut-être des opérations critiques
- ❓ Sans validation : vous ne détectez pas les problèmes
- ❓ Comment tester sans risque ?

**Solution** : L'action `warn` !

### Analogie avec un contrôle routier

**Action `error`** = Barrière physique
- Le véhicule non conforme ne peut **pas passer**
- Blocage immédiat et visible
- Garantie que rien de non conforme ne passe

**Action `warn`** = Caméra de surveillance
- Le véhicule non conforme **peut passer**
- L'infraction est **enregistrée** dans les logs
- Permet l'observation et l'analyse sans bloquer le trafic

---

## 🚫 Action `error` (par défaut)

### Définition

L'action **error** **refuse** toute opération qui violerait les règles de validation et retourne une erreur à l'application.

### Comportement

```javascript
db.createCollection("produits", {
  validator: { $jsonSchema: { /* règles */ } },
  validationAction: "error"  // Par défaut, peut être omis
})
```

**Règle** : Si le document ne respecte pas les règles → **Opération rejetée**

### Exemple pratique

```javascript
// Créer une collection avec action "error"
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["numero", "client", "montant"],
      properties: {
        numero: {
          bsonType: "string",
          pattern: "^CMD-[0-9]{6}$"
        },
        client: {
          bsonType: "string",
          minLength: 2
        },
        montant: {
          bsonType: "double",
          minimum: 0
        }
      }
    }
  },
  validationAction: "error"
})

// ✅ Insertion valide
db.commandes.insertOne({
  numero: "CMD-123456",
  client: "Dupont",
  montant: 150.00
})
// Résultat : { acknowledged: true, insertedId: ObjectId("...") }

// ❌ Insertion invalide : numéro mal formaté
db.commandes.insertOne({
  numero: "12345",  // Ne correspond pas au pattern
  client: "Martin",
  montant: 200.00
})
// Résultat : MongoServerError: Document failed validation
// Additional info: {
//   failingDocumentId: ObjectId("..."),
//   details: {
//     operatorName: '$jsonSchema',
//     schemaRulesNotSatisfied: [...]
//   }
// }
// L'opération est REJETÉE

// ❌ Insertion invalide : montant négatif
db.commandes.insertOne({
  numero: "CMD-789012",
  client: "Bernard",
  montant: -50.00  // Montant négatif interdit
})
// Résultat : MongoServerError: Document failed validation
// L'opération est REJETÉE
```

### Message d'erreur typique

Quand une validation échoue en mode `error`, MongoDB retourne un message détaillé :

```javascript
MongoServerError: Document failed validation
{
  "failingDocumentId": ObjectId("507f1f77bcf86cd799439011"),
  "details": {
    "operatorName": "$jsonSchema",
    "schemaRulesNotSatisfied": [
      {
        "operatorName": "properties",
        "propertiesNotSatisfied": [
          {
            "propertyName": "montant",
            "details": [
              {
                "operatorName": "minimum",
                "specifiedAs": { "minimum": 0 },
                "reason": "comparison failed",
                "consideredValue": -50
              }
            ]
          }
        ]
      }
    ]
  }
}
```

### Quand utiliser l'action `error`

- ✅ **Production avec données critiques** - Applications bancaires, santé, etc.
- ✅ **Après phase de test** - Quand vous êtes sûr des règles de validation
- ✅ **Intégrité essentielle** - Quand la cohérence est non négociable
- ✅ **Nouvelles applications** - Pas de données historiques à gérer
- ✅ **Environnements contrôlés** - Développement et staging

### Avantages et inconvénients

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Garantie absolue de conformité | ❌ Peut bloquer l'application |
| ✅ Détection immédiate des erreurs | ❌ Nécessite des règles bien testées |
| ✅ Feedback instantané aux développeurs | ❌ Difficile à déployer sur l'existant |
| ✅ Empêche la pollution des données | ❌ Risque de casser des fonctionnalités |
| ✅ Simplicité conceptuelle | ❌ Pas de phase d'observation |

---

## ⚠️ Action `warn`

### Définition

L'action **warn** **accepte** toutes les opérations, même celles qui violent les règles, mais **enregistre un avertissement** dans les logs MongoDB.

### Comportement

```javascript
db.createCollection("produits", {
  validator: { $jsonSchema: { /* règles */ } },
  validationAction: "warn"
})
```

**Règle** : Si le document ne respecte pas les règles → **Opération acceptée + Log d'avertissement**

### Exemple pratique

```javascript
// Créer une collection avec action "warn"
db.createCollection("articles", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["titre", "contenu"],
      properties: {
        titre: {
          bsonType: "string",
          minLength: 5,
          maxLength: 100
        },
        contenu: {
          bsonType: "string",
          minLength: 10
        },
        categorie: {
          enum: ["tech", "sport", "culture", "economie"]
        }
      }
    }
  },
  validationAction: "warn"
})

// ✅ Insertion valide
db.articles.insertOne({
  titre: "Introduction à MongoDB",
  contenu: "MongoDB est une base de données NoSQL..."
})
// Résultat : { acknowledged: true, insertedId: ObjectId("...") }
// Pas de log d'avertissement

// ⚠️ Insertion invalide : ACCEPTÉE mais avec avertissement
db.articles.insertOne({
  titre: "Test",  // Trop court (< 5 caractères)
  contenu: "Court"  // Trop court (< 10 caractères)
})
// Résultat : { acknowledged: true, insertedId: ObjectId("...") }
// L'insertion RÉUSSIT !
// Un avertissement est enregistré dans les logs

// ⚠️ Insertion invalide : catégorie incorrecte
db.articles.insertOne({
  titre: "Article de politique",
  contenu: "Contenu de l'article politique...",
  categorie: "politique"  // Pas dans l'enum
})
// Résultat : { acknowledged: true, insertedId: ObjectId("...") }
// L'insertion RÉUSSIT !
// Un avertissement est enregistré dans les logs
```

### Entrée de log typique

Quand une validation échoue en mode `warn`, MongoDB écrit dans les logs :

```json
{
  "t": {
    "$date": "2025-01-15T10:30:45.123Z"
  },
  "s": "W",
  "c": "STORAGE",
  "id": 20294,
  "ctx": "conn123",
  "msg": "Document would fail validation",
  "attr": {
    "namespace": "maBase.articles",
    "document": {
      "_id": ObjectId("507f1f77bcf86cd799439011"),
      "titre": "Test",
      "contenu": "Court"
    },
    "errInfo": {
      "failingDocumentId": ObjectId("507f1f77bcf86cd799439011"),
      "details": {
        "operatorName": "$jsonSchema",
        "schemaRulesNotSatisfied": [
          {
            "operatorName": "properties",
            "propertiesNotSatisfied": [
              {
                "propertyName": "titre",
                "details": [
                  {
                    "operatorName": "minLength",
                    "specifiedAs": { "minLength": 5 },
                    "reason": "specified string length was not satisfied",
                    "consideredValue": "Test"
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  }
}
```

### Consulter les logs d'avertissement

**Sur un serveur MongoDB local** :

```bash
# Voir les logs en temps réel
tail -f /var/log/mongodb/mongod.log | grep "Document would fail validation"

# Rechercher tous les avertissements de validation
grep "Document would fail validation" /var/log/mongodb/mongod.log
```

**Depuis mongosh** :

```javascript
// Activer le profiler pour capturer les opérations
db.setProfilingLevel(2)

// Consulter le profiler
db.system.profile.find({
  ns: "maBase.articles",
  op: "insert"
}).pretty()
```

**Sur MongoDB Atlas** :
- Interface web → Onglet "Logs"
- Filtrer par "validation"

### Quand utiliser l'action `warn`

- ✅ **Phase de test** - Tester de nouvelles règles sans impact
- ✅ **Migration** - Observer l'impact avant activation strict
- ✅ **Analyse des données** - Identifier les documents non conformes
- ✅ **Déploiement progressif** - Activation en douceur
- ✅ **Audit et monitoring** - Surveillance sans blocage
- ✅ **Collections existantes** - Comprendre l'état actuel

### Avantages et inconvénients

| Avantages | Inconvénients |
|-----------|---------------|
| ✅ Aucun blocage d'opérations | ❌ Pas de garantie de conformité |
| ✅ Observation sans risque | ❌ Documents non conformes acceptés |
| ✅ Déploiement sans casse | ❌ Nécessite surveillance des logs |
| ✅ Feedback sans interruption | ❌ Peut donner faux sentiment de sécurité |
| ✅ Idéal pour tests et migrations | ❌ Données incohérentes possibles |

---

## 🔄 Comparaison des deux actions

### Tableau comparatif

| Aspect | `error` | `warn` |
|--------|---------|--------|
| **Document valide** | ✅ Accepté | ✅ Accepté |
| **Document invalide** | ❌ Rejeté | ⚠️ Accepté + Log |
| **Retour à l'application** | Erreur | Succès |
| **Garantie de conformité** | Totale | Aucune |
| **Impact sur application** | Possible | Aucun |
| **Visibilité des problèmes** | Immédiate | Dans les logs |
| **Cas d'usage principal** | Production | Tests / Monitoring |

### Schéma visuel

```
ACTION ERROR
════════════

Document valide   → [VALIDATION] → ✅ ACCEPTÉ
Document invalide → [VALIDATION] → ❌ REJETÉ (erreur retournée)


ACTION WARN
═══════════

Document valide   → [VALIDATION] → ✅ ACCEPTÉ
Document invalide → [VALIDATION] → ⚠️ ACCEPTÉ (log d'avertissement)
```

### Exemple comparatif complet

```javascript
// Document qui va être testé (invalide)
const documentInvalide = {
  nom: "P",  // Trop court (minLength: 2)
  prix: -10  // Négatif (minimum: 0)
}

// Règles de validation
const validationRules = {
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "prix"],
    properties: {
      nom: {
        bsonType: "string",
        minLength: 2
      },
      prix: {
        bsonType: "double",
        minimum: 0
      }
    }
  }
}

// ────────────────────────────────────────────
// SCÉNARIO 1 : ACTION ERROR
// ────────────────────────────────────────────

db.createCollection("produits_error", {
  validator: validationRules,
  validationAction: "error"
})

try {
  db.produits_error.insertOne(documentInvalide)
  console.log("Document inséré ✅")
} catch (error) {
  console.log("Document rejeté ❌")
  console.log("Erreur:", error.message)
}
// Sortie :
// Document rejeté ❌
// Erreur: Document failed validation

// Vérifier le nombre de documents
db.produits_error.countDocuments()
// Résultat : 0 (le document n'a PAS été inséré)


// ────────────────────────────────────────────
// SCÉNARIO 2 : ACTION WARN
// ────────────────────────────────────────────

db.createCollection("produits_warn", {
  validator: validationRules,
  validationAction: "warn"
})

try {
  const result = db.produits_warn.insertOne(documentInvalide)
  console.log("Document inséré ✅")
  console.log("ID:", result.insertedId)
} catch (error) {
  console.log("Document rejeté ❌")
}
// Sortie :
// Document inséré ✅
// ID: ObjectId("...")

// Vérifier le nombre de documents
db.produits_warn.countDocuments()
// Résultat : 1 (le document a été inséré !)

// Un avertissement est dans les logs MongoDB
```

---

## 🔧 Changer l'action de validation

### Sur une collection existante

Utilisez la commande `collMod` :

```javascript
// Passer en mode error
db.runCommand({
  collMod: "maCollection",
  validator: { $jsonSchema: { /* règles */ } },
  validationAction: "error"
})

// Passer en mode warn
db.runCommand({
  collMod: "maCollection",
  validator: { $jsonSchema: { /* règles */ } },
  validationAction: "warn"
})
```

### Consulter l'action actuelle

```javascript
// Voir la configuration de validation
db.getCollectionInfos({ name: "maCollection" })[0].options.validationAction
// Résultat : "error" ou "warn"
```

### Exemple de modification

```javascript
// État initial : error
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["numero", "montant"],
      properties: {
        numero: { bsonType: "string" },
        montant: { bsonType: "double", minimum: 0 }
      }
    }
  },
  validationAction: "error"
})

// Vérifier l'action
db.getCollectionInfos({ name: "commandes" })[0].options.validationAction
// Résultat : "error"

// Passer en warn pour tester de nouvelles règles
db.runCommand({
  collMod: "commandes",
  validationAction: "warn"
})

// Vérifier à nouveau
db.getCollectionInfos({ name: "commandes" })[0].options.validationAction
// Résultat : "warn"
```

---

## 📋 Stratégies de déploiement

### Stratégie 1 : Déploiement progressif (recommandée)

**Phase 1 : Mode warn avec surveillance**

```javascript
// Étape 1 : Activer en mode warn
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix"],
      properties: {
        nom: { bsonType: "string", minLength: 2 },
        prix: { bsonType: "double", minimum: 0 }
      }
    }
  },
  validationAction: "warn"
})

// Étape 2 : Laisser tourner 1-2 semaines
// Observer les logs pour identifier les problèmes

// Étape 3 : Analyser les avertissements
// Script pour compter les violations
grep "Document would fail validation" /var/log/mongodb/mongod.log | wc -l
```

**Phase 2 : Correction des sources**

```javascript
// Identifier les documents problématiques
db.produits.find({
  $or: [
    { nom: { $type: "string", $not: { $regex: /.{2,}/ } } },
    { prix: { $lt: 0 } }
  ]
})

// Corriger les documents existants
db.produits.updateMany(
  { nom: { $exists: true, $type: "string" }, $expr: { $lt: [{ $strLenCP: "$nom" }, 2] } },
  [{ $set: { nom: { $concat: ["$nom", "_"] } } }]
)

db.produits.updateMany(
  { prix: { $lt: 0 } },
  { $set: { prix: 0 } }
)
```

**Phase 3 : Passage en mode error**

```javascript
// Une fois que les logs sont propres pendant plusieurs jours
db.runCommand({
  collMod: "produits",
  validationAction: "error"
})

// Surveiller les premières heures après activation
```

### Stratégie 2 : Mode warn permanent pour audit

Pour certaines collections, `warn` peut être un choix permanent :

```javascript
// Collection de logs/événements
// On veut tracer les anomalies sans bloquer
db.createCollection("logs_application", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["timestamp", "level", "message"],
      properties: {
        timestamp: { bsonType: "date" },
        level: { enum: ["DEBUG", "INFO", "WARN", "ERROR"] },
        message: { bsonType: "string" }
      }
    }
  },
  validationAction: "warn"  // Permanent : on ne bloque jamais les logs
})
```

### Stratégie 3 : Mode error dès le départ

Pour les nouvelles collections critiques :

```javascript
// Nouvelle fonctionnalité : paiements
// Validation stricte dès le départ
db.createCollection("paiements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["montant", "devise", "statut", "timestamp"],
      properties: {
        montant: {
          bsonType: "decimal",
          minimum: 0.01,
          description: "Montant en devise"
        },
        devise: {
          enum: ["EUR", "USD", "GBP"],
          description: "Code devise ISO"
        },
        statut: {
          enum: ["en_attente", "valide", "refuse", "annule"],
          description: "Statut du paiement"
        },
        timestamp: {
          bsonType: "date",
          description: "Date du paiement"
        }
      }
    }
  },
  validationAction: "error"  // Strict dès le début
})
```

---

## 🎯 Cas d'usage réels

### Cas 1 : Test de nouvelles contraintes

**Situation** :
- Collection existante avec 100 000 produits
- Ajout d'une nouvelle contrainte : le prix ne peut pas dépasser 10 000 €
- Incertitude sur l'impact

**Solution** :

```javascript
// 1. Ajouter la contrainte en mode warn
db.runCommand({
  collMod: "produits",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        prix: {
          bsonType: "double",
          maximum: 10000  // Nouvelle contrainte
        }
      }
    }
  },
  validationAction: "warn"
})

// 2. Observer pendant 1 semaine

// 3. Analyser les logs
// Si peu d'avertissements → OK pour passer en error
// Si beaucoup → revoir la contrainte ou corriger les données

// 4. Compter les produits qui dépassent
db.produits.countDocuments({ prix: { $gt: 10000 } })
// Résultat : 23 produits

// 5. Décision : corriger ces 23 produits puis passer en error
```

### Cas 2 : Migration d'API externe

**Situation** :
- API externe change de format de données
- Ancien format : `{ telephone: "0612345678" }`
- Nouveau format : `{ contacts: { mobile: "0612345678" } }`
- Transition progressive nécessaire

**Solution** :

```javascript
// 1. Validation du nouveau format en mode warn
db.runCommand({
  collMod: "clients",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        contacts: {
          bsonType: "object",
          properties: {
            mobile: { bsonType: "string" }
          }
        }
      }
    }
  },
  validationAction: "warn"
})

// 2. Application gère les deux formats pendant la transition

// 3. Migration progressive des données
db.clients.updateMany(
  { telephone: { $exists: true }, contacts: { $exists: false } },
  [{
    $set: {
      contacts: {
        mobile: "$telephone"
      }
    }
  }]
)

// 4. Une fois migration complète, passer en error
db.runCommand({
  collMod: "clients",
  validationAction: "error"
})
```

### Cas 3 : Monitoring de la qualité des données

**Situation** :
- Système de collecte de données IoT
- Certains capteurs envoient parfois des valeurs aberrantes
- Ne pas bloquer la collecte, mais identifier les anomalies

**Solution** :

```javascript
// Validation en mode warn permanent
db.createCollection("mesures_iot", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["capteur_id", "timestamp", "valeur"],
      properties: {
        capteur_id: { bsonType: "string" },
        timestamp: { bsonType: "date" },
        valeur: {
          bsonType: "double",
          minimum: -50,   // Température minimale raisonnable
          maximum: 150    // Température maximale raisonnable
        }
      }
    }
  },
  validationAction: "warn"  // Permanent
})

// Script d'analyse quotidien des anomalies détectées
// grep "Document would fail validation" logs | analyse_anomalies.sh
```

### Cas 4 : Feature flag pour validation

**Situation** :
- Déploiement graduel d'une nouvelle fonctionnalité
- Activation progressive selon les utilisateurs
- Validation uniquement pour les nouveaux utilisateurs

**Solution** :

```javascript
// Phase 1 : warn pour tout le monde
db.runCommand({
  collMod: "utilisateurs",
  validator: { $jsonSchema: { /* nouvelle structure */ } },
  validationAction: "warn"
})

// Phase 2 : error pour 10% des utilisateurs (dans application)
// L'application vérifie avant insertion et rejette côté app

// Phase 3 : error pour 50% des utilisateurs

// Phase 4 : error pour tous
db.runCommand({
  collMod: "utilisateurs",
  validationAction: "error"
})
```

---

## 💡 Bonnes pratiques

### 1. Toujours commencer par `warn` en production

```javascript
// ✅ Approche sûre
db.runCommand({
  collMod: "collection_production",
  validator: { $jsonSchema: { /* règles */ } },
  validationAction: "warn"  // Observer d'abord
})

// Observer pendant quelques jours/semaines

// Puis passer en error
db.runCommand({
  collMod: "collection_production",
  validationAction: "error"
})
```

### 2. Automatiser la surveillance des logs en mode `warn`

```bash
#!/bin/bash
# Script de monitoring des avertissements de validation

LOG_FILE="/var/log/mongodb/mongod.log"
ALERT_THRESHOLD=100

# Compter les avertissements dans la dernière heure
WARNINGS=$(grep "Document would fail validation" "$LOG_FILE" | \
           grep "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H')" | \
           wc -l)

if [ "$WARNINGS" -gt "$ALERT_THRESHOLD" ]; then
  echo "⚠️ ALERTE : $WARNINGS avertissements de validation détectés !"
  # Envoyer notification (email, Slack, etc.)
fi
```

### 3. Documenter la raison du mode `warn`

```javascript
// ✅ Avec documentation
db.runCommand({
  collMod: "ancienne_collection",
  validator: { $jsonSchema: { /* règles */ } },
  validationAction: "warn"  // Mode warn jusqu'au 31/01/2025
  // Raison : Migration progressive depuis ancien format
  // Passer en error après correction des 5000 documents restants
  // Ticket : PROJ-1234
})
```

### 4. Utiliser des métriques pour décider

```javascript
// Script pour évaluer le taux de conformité
function tauxConformite(collectionName, validator) {
  const total = db[collectionName].countDocuments()

  // Compter manuellement les documents non conformes
  // (car en mode warn, ils sont tous dans la collection)
  const nonConformes = db[collectionName].countDocuments({
    /* critères de non-conformité */
  })

  const tauxConformite = ((total - nonConformes) / total * 100).toFixed(2)

  console.log(`Collection: ${collectionName}`)
  console.log(`Total documents: ${total}`)
  console.log(`Non conformes: ${nonConformes}`)
  console.log(`Taux de conformité: ${tauxConformite}%`)

  if (tauxConformite >= 99) {
    console.log("✅ Prêt pour passage en mode error")
  } else {
    console.log("⚠️ Encore du travail avant mode error")
  }
}

tauxConformite("produits", validator)
```

### 5. Combiner avec `validationLevel` pour maximum de contrôle

```javascript
// Configuration très progressive
db.runCommand({
  collMod: "clients",
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "moderate",   // Valide uniquement les nouveaux
  validationAction: "warn"       // Sans bloquer
})

// Permet :
// - Nouveaux documents : conformes idéalement, mais pas bloqués si erreur
// - Anciens documents : pas de validation du tout
// Idéal pour phase de test sur production
```

---

## ⚠️ Pièges à éviter

### 1. Oublier de surveiller les logs en mode `warn`

```javascript
// ❌ Activer warn et ne jamais vérifier les logs
db.runCommand({
  collMod: "produits",
  validationAction: "warn"
})
// Résultat : aucune valeur ajoutée, faux sentiment de sécurité

// ✅ Activer warn avec monitoring
db.runCommand({
  collMod: "produits",
  validationAction: "warn"
})
// + Mettre en place alertes sur les logs
// + Analyser régulièrement les violations
```

### 2. Garder `warn` trop longtemps

```javascript
// ❌ Mode warn depuis 6 mois sans action
// Les développeurs s'habituent aux violations

// ✅ Plan clair avec dates
// Warn du 01/01 au 31/01 : observation
// Février : corrections
// Mars : passage en error
```

### 3. Passer en `error` sans tests préalables

```javascript
// ❌ DANGER : Passer directement en error sans warn
db.runCommand({
  collMod: "collection_production",
  validator: { $jsonSchema: { /* nouvelles règles strictes */ } },
  validationAction: "error"  // Sans phase de test !
})
// Peut casser l'application en production !

// ✅ Toujours tester en warn d'abord
```

### 4. Ne pas communiquer avec l'équipe

Mode `warn` → Mode `error` doit être communiqué à toute l'équipe :
- Développeurs backend
- Développeurs frontend
- Équipe QA
- DevOps / SRE

### 5. Ignorer les avertissements récurrents

Si les mêmes violations apparaissent constamment en mode `warn` :
- Soit corriger le code source
- Soit revoir les règles de validation
- Ne pas laisser pourrir

---

## 🎓 Résumé

| Aspect | `error` | `warn` |
|--------|---------|--------|
| **Comportement** | Rejette documents invalides | Accepte tout + log |
| **Impact application** | Peut bloquer | Aucun blocage |
| **Visibilité** | Erreur immédiate | Dans les logs |
| **Garantie conformité** | Totale | Aucune |
| **Cas d'usage** | Production après tests | Tests, monitoring, migration |
| **Recommandé pour** | Collections critiques validées | Phase d'observation |
| **Risques** | Casser l'application | Faux sentiment de sécurité |

### Points clés à retenir

- ✅ **`error`** = Rejette les documents invalides (garantie de conformité)
- ✅ **`warn`** = Accepte tout mais enregistre les violations (observation sans risque)
- ✅ Toujours commencer par `warn` sur collections existantes
- ✅ Surveiller les logs activement en mode `warn`
- ✅ Passer en `error` après analyse et corrections
- ✅ Combiner avec `validationLevel` pour flexibilité maximale
- ✅ Documenter et communiquer les changements

### Progression recommandée

```
1. warn + moderate  → Observation douce
2. warn + strict    → Observation stricte
3. error + moderate → Validation modérée
4. error + strict   → Validation maximale
```

---

## 📚 Dans la prochaine section

Dans la section suivante (7.6), nous verrons comment **modifier les règles de validation** sur des collections existantes de manière sûre et progressive.

---


⏭️ [Modification des règles de validation](/07-validation-des-schemas/06-modification-regles.md)
