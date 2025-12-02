🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 7 : Validation des Schémas

## 📚 Vue d'ensemble du chapitre

Bienvenue dans le chapitre sur la **validation des schémas** dans MongoDB ! Ce chapitre est essentiel pour garantir la **qualité et la cohérence** de vos données, tout en conservant la flexibilité qui fait la force de MongoDB.

---

## 🎯 Objectifs d'apprentissage

À la fin de ce chapitre, vous serez capable de :

- ✅ Comprendre pourquoi et quand utiliser la validation de schéma
- ✅ Créer des règles de validation avec `$jsonSchema`
- ✅ Valider tous les types de données BSON
- ✅ Gérer les champs obligatoires et les dépendances entre champs
- ✅ Utiliser `$expr` pour des validations complexes
- ✅ Choisir entre les modes `strict` et `moderate`
- ✅ Choisir entre les actions `error` et `warn`
- ✅ Modifier les règles de validation en toute sécurité
- ✅ Appliquer les bonnes pratiques de validation en production

---

## 🤔 Pourquoi la validation de schéma ?

### Le paradoxe de MongoDB

MongoDB est connu pour sa **flexibilité de schéma** : vous pouvez insérer des documents avec des structures différentes dans la même collection. C'est à la fois une force et un risque potentiel.

**Sans validation** :
```javascript
// Document 1 : Structure complète
{
  nom: "Dupont",
  email: "dupont@example.com",
  age: 30
}

// Document 2 : Structure différente
{
  nom: "Martin",
  telephone: "0612345678"
}

// Document 3 : Même vide !
{
  note: "Document de test"
}
```

**Conséquences** :
- ❌ Données incohérentes
- ❌ Bugs difficiles à détecter
- ❌ Requêtes qui échouent de manière imprévisible
- ❌ Difficultés de maintenance

**Avec validation** :
```javascript
// Règles claires et appliquées automatiquement
{
  validator: {
    $jsonSchema: {
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string", minLength: 2 },
        email: { bsonType: "string", pattern: "^.+@.+\\..+$" },
        age: { bsonType: "int", minimum: 0, maximum: 150 }
      }
    }
  }
}
```

**Résultat** :
- ✅ Cohérence garantie
- ✅ Détection immédiate des erreurs
- ✅ Code applicatif plus simple
- ✅ Maintenance facilitée

### Validation côté base vs côté application

**Question courante** : Pourquoi valider dans MongoDB si je valide déjà dans mon application ?

**Réponse** : Les deux sont complémentaires !

| Validation application | Validation MongoDB |
|------------------------|-------------------|
| Premier niveau de défense | Dernier rempart |
| Messages d'erreur conviviaux | Garantie d'intégrité |
| Logique métier complexe | Règles de structure |
| Peut être contournée | Impossible à contourner |
| Une application = une validation | Toutes les applications = mêmes règles |

**Meilleure pratique** : Validez dans les deux !

---

## 📖 Structure du chapitre

Ce chapitre est organisé en **10 sections** qui vous guideront progressivement du concept de base aux techniques avancées.

### 📍 Section 7.1 - Introduction à la validation de schéma

**Ce que vous apprendrez** :
- Les concepts fondamentaux de la validation
- Quand et pourquoi utiliser la validation
- Vue d'ensemble des outils disponibles

**Niveau** : 🟢 Débutant

---

### 📍 Section 7.2 - JSON Schema dans MongoDB

**Ce que vous apprendrez** :
- Le standard JSON Schema
- Les différences entre JSON et BSON
- La syntaxe de base de `$jsonSchema`
- Structure d'un schéma de validation

**Niveau** : 🟢 Débutant

---

### 📍 Section 7.3 - Règles de validation ($jsonSchema)

**Ce que vous apprendrez** :
- Toutes les règles de validation disponibles
- Règles pour strings, nombres, tableaux, objets
- Opérateurs logiques (`oneOf`, `anyOf`, `allOf`, `not`)
- Exemples pratiques pour chaque règle

**Niveau** : 🟡 Intermédiaire

---

### 📍 Section 7.4 - Niveaux de validation (strict, moderate)

**Ce que vous apprendrez** :
- Différence entre `strict` et `moderate`
- Quand utiliser chaque niveau
- Stratégies de migration
- Impact sur les documents existants

**Niveau** : 🟡 Intermédiaire

---

### 📍 Section 7.5 - Actions de validation (error, warn)

**Ce que vous apprendrez** :
- Différence entre `error` et `warn`
- Mode observation vs mode blocage
- Consultation des logs d'avertissement
- Stratégies de déploiement progressif

**Niveau** : 🟡 Intermédiaire

---

### 📍 Section 7.6 - Modification des règles de validation

**Ce que vous apprendrez** :
- Comment modifier un schéma existant en toute sécurité
- Consulter les règles actuelles
- Ajouter ou retirer des règles
- Plans de rollback et sauvegarde
- Stratégies de migration

**Niveau** : 🟡 Intermédiaire

---

### 📍 Section 7.7 - Validation des types de données

**Ce que vous apprendrez** :
- Validation détaillée de chaque type BSON
- `string`, `int`, `long`, `double`, `decimal`, `bool`, `date`, `objectId`
- Tableaux et objets imbriqués
- Accepter plusieurs types
- Patterns de validation courants

**Niveau** : 🟡 Intermédiaire

---

### 📍 Section 7.8 - Validation des champs obligatoires

**Ce que vous apprendrez** :
- Utilisation de `required`
- Dépendances entre champs avec `dependencies`
- Champs obligatoires conditionnels
- Champs obligatoires dans objets imbriqués
- Stratégies de gestion

**Niveau** : 🟡 Intermédiaire

---

### 📍 Section 7.9 - Validation personnalisée avec $expr

**Ce que vous apprendrez** :
- Quand et pourquoi utiliser `$expr`
- Comparaison entre champs
- Calculs et expressions arithmétiques
- Validations temporelles avec `$$NOW`
- Logique conditionnelle complexe
- Combiner `$jsonSchema` et `$expr`

**Niveau** : 🔴 Avancé

---

### 📍 Section 7.10 - Bonnes pratiques de validation

**Ce que vous apprendrez** :
- Principes généraux de conception
- Organisation et maintenance des schémas
- Performance et optimisation
- Sécurité et qualité des données
- Déploiement et migration
- Anti-patterns à éviter
- Checklist complète

**Niveau** : 🔴 Avancé

---

## 🛤️ Parcours d'apprentissage recommandé

### Pour les débutants

**Parcours minimal** (4-6 heures) :
1. Section 7.1 - Introduction
2. Section 7.2 - JSON Schema
3. Section 7.3 - Règles de validation (survol)
4. Section 7.7 - Types de données (focus sur les types courants)

**Objectif** : Comprendre les bases et créer des validations simples

### Pour les développeurs

**Parcours complet** (8-12 heures) :
1. Toutes les sections dans l'ordre
2. Focus particulier sur :
   - Section 7.4 - Niveaux de validation
   - Section 7.5 - Actions de validation
   - Section 7.6 - Modification des règles
   - Section 7.10 - Bonnes pratiques

**Objectif** : Maîtriser la validation pour des applications en production

### Pour les architectes et DevOps

**Parcours expert** (12-16 heures) :
1. Lecture complète de toutes les sections
2. Focus avancé sur :
   - Section 7.9 - Validation avec $expr
   - Section 7.10 - Bonnes pratiques (en profondeur)
3. Étude des stratégies de migration
4. Mise en place de processus de validation

**Objectif** : Concevoir des architectures de validation robustes et évolutives

---

## 🎨 Concepts clés du chapitre

### 1. Validation avec $jsonSchema

Le cœur de la validation MongoDB. Permet de définir :
- Types de données
- Champs obligatoires
- Formats (via regex)
- Contraintes (min/max, longueur, enum)
- Structure des objets et tableaux

### 2. Niveaux de validation

**`strict`** : Valide tous les documents (nouveaux et existants)
**`moderate`** : Valide uniquement les nouveaux documents

### 3. Actions de validation

**`error`** : Rejette les documents invalides
**`warn`** : Accepte tout mais enregistre dans les logs

### 4. Validation avancée avec $expr

Utilise le langage d'agrégation pour :
- Comparer des champs entre eux
- Effectuer des calculs
- Créer des validations conditionnelles
- Valider avec la date actuelle (`$$NOW`)

---

## 🔧 Outils et commandes principales

### Commandes essentielles

```javascript
// Créer une collection avec validation
db.createCollection("nom", {
  validator: { $jsonSchema: { /* règles */ } },
  validationLevel: "strict",
  validationAction: "error"
})

// Modifier la validation
db.runCommand({
  collMod: "nom",
  validator: { $jsonSchema: { /* nouvelles règles */ } },
  validationLevel: "strict",
  validationAction: "error"
})

// Consulter la validation actuelle
db.getCollectionInfos({ name: "nom" })
```

### Structure de base d'un schéma

```javascript
{
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["champ1", "champ2"],
      properties: {
        champ1: {
          bsonType: "string",
          description: "Description du champ"
        },
        champ2: {
          bsonType: "int",
          minimum: 0
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
}
```

---

## 💡 Points importants à retenir

### ✅ Ce que la validation FAIT

- Garantit la **structure** des documents
- Valide les **types de données**
- Vérifie les **formats** (email, téléphone, etc.)
- Impose des **contraintes** (min/max, longueur)
- Compare des **champs entre eux** (avec $expr)
- Protège contre les **erreurs de programmation**

### ❌ Ce que la validation NE FAIT PAS

- Ne remplace pas la validation **côté application**
- Ne valide pas les **références entre collections**
- Ne garantit pas la **cohérence transactionnelle complexe**
- N'empêche pas tous les **bugs logiques**
- Ne valide pas les **permissions utilisateur**

### 🎯 Quand utiliser la validation

**✅ Toujours utiliser pour** :
- Champs critiques (identifiants, clés étrangères)
- Types de données importants
- Formats sensibles (email, numéros)
- Données financières ou légales

**⚠️ Utiliser avec précaution pour** :
- Collections avec données historiques
- Prototypes et phases de développement initial
- Collections avec structure très variable

**❌ Ne pas utiliser pour** :
- Logs et données temporaires
- Collections de test
- Données dont la structure doit évoluer rapidement

---

## 🚀 Progression dans le chapitre

```
Débutant                 Intermédiaire                    Avancé
│                        │                                │
├─ 7.1 Introduction      ├─ 7.4 Niveaux validation       ├─ 7.9 Validation $expr
├─ 7.2 JSON Schema       ├─ 7.5 Actions validation       └─ 7.10 Bonnes pratiques
└─ 7.3 Règles (base)     ├─ 7.6 Modification règles
                         ├─ 7.7 Types de données
                         └─ 7.8 Champs obligatoires
```

---

## 📊 Matrice de complexité

| Section | Difficulté | Temps estimé | Prérequis |
|---------|-----------|--------------|-----------|
| 7.1 | 🟢 Facile | 30 min | Chapitres 1-2 |
| 7.2 | 🟢 Facile | 45 min | Section 7.1 |
| 7.3 | 🟡 Moyen | 2 heures | Section 7.2 |
| 7.4 | 🟡 Moyen | 1 heure | Sections 7.1-7.3 |
| 7.5 | 🟡 Moyen | 1 heure | Sections 7.1-7.3 |
| 7.6 | 🟡 Moyen | 1h30 | Sections 7.1-7.5 |
| 7.7 | 🟡 Moyen | 2 heures | Section 7.2 |
| 7.8 | 🟡 Moyen | 1h30 | Sections 7.2-7.3 |
| 7.9 | 🔴 Difficile | 2h30 | Sections 7.1-7.8 |
| 7.10 | 🔴 Difficile | 2 heures | Toutes les sections |

**Temps total estimé** : 14-16 heures

---

## 🎓 Compétences acquises

Après avoir terminé ce chapitre, vous aurez acquis les compétences suivantes :

### Niveau Débutant
- ✅ Comprendre le concept de validation de schéma
- ✅ Créer des validations simples avec `$jsonSchema`
- ✅ Valider les types de données de base
- ✅ Définir des champs obligatoires

### Niveau Intermédiaire
- ✅ Utiliser toutes les règles de validation disponibles
- ✅ Choisir le bon niveau et la bonne action de validation
- ✅ Modifier les schémas en toute sécurité
- ✅ Valider des structures complexes (objets imbriqués, tableaux)
- ✅ Gérer les dépendances entre champs

### Niveau Avancé
- ✅ Créer des validations personnalisées avec `$expr`
- ✅ Comparer des champs et effectuer des calculs
- ✅ Mettre en place des stratégies de migration complexes
- ✅ Optimiser les performances de validation
- ✅ Appliquer les meilleures pratiques en production

---

## 🔗 Liens avec les autres chapitres

### Prérequis recommandés

- **Chapitre 2** - Fondamentaux de MongoDB : Comprendre la structure des documents
- **Chapitre 3** - Requêtes et Filtres : Connaître les opérateurs de base

### Chapitres suivants

- **Chapitre 8** - Transactions : La validation garantit la cohérence dans les transactions
- **Chapitre 11** - Sécurité : La validation fait partie de la stratégie de sécurité
- **Chapitre 21** - Bonnes pratiques : Intégration de la validation dans vos processus

---

## 🎯 Cas d'usage couverts

Ce chapitre couvre les cas d'usage suivants à travers des exemples pratiques :

- 🛒 **E-commerce** : Validation de produits, commandes, paniers
- 👤 **Gestion utilisateurs** : Profils, inscriptions, authentification
- 💰 **Finance** : Transactions, paiements, factures
- 📅 **Réservations** : Événements, locations, rendez-vous
- 🏢 **Entreprise** : Employés, congés, documents
- 🌐 **IoT** : Capteurs, mesures, données temps réel

---

## 💼 Applications pratiques

### Pour les développeurs

- Protéger votre application contre les bugs de validation
- Réduire le code de validation côté application
- Garantir la cohérence entre plusieurs services
- Faciliter le débogage avec des erreurs claires

### Pour les équipes DevOps

- Standardiser les structures de données
- Faciliter les migrations de schéma
- Améliorer la qualité des données en production
- Réduire les incidents liés aux données

### Pour les architectes

- Concevoir des modèles de données robustes
- Planifier l'évolution des schémas
- Assurer la compatibilité entre versions
- Optimiser les performances de validation

---

## 📝 Conseils pour tirer le meilleur parti de ce chapitre

### 1. Pratiquez progressivement

Ne cherchez pas à tout apprendre d'un coup. Commencez par les sections 7.1 à 7.3, puis avancez progressivement.

### 2. Testez tous les exemples

Chaque section contient de nombreux exemples. Testez-les dans votre environnement MongoDB pour bien comprendre.

### 3. Créez vos propres schémas

Après chaque section, essayez de créer un schéma de validation pour vos propres cas d'usage.

### 4. Consultez la documentation officielle

Ce chapitre est complet, mais la documentation MongoDB officielle contient des détails supplémentaires.

### 5. Commencez en mode "warn"

Quand vous testez de nouvelles validations, utilisez toujours le mode "warn" avant "error".

---

## 🚦 Prêt à commencer ?

Vous êtes maintenant prêt à plonger dans le monde de la validation de schéma MongoDB !

**Prochaine étape** : Section 7.1 - Introduction à la validation de schéma

---

**Bon apprentissage ! 🎓**

---

## Navigation


⏭️ [Introduction à la validation de schéma](/07-validation-des-schemas/01-introduction-validation.md)
