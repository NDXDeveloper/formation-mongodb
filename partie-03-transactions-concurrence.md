🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 3 : Transactions et Concurrence (Avancé)

## 🎯 Garanties de cohérence dans les systèmes distribués

Vous avez maîtrisé la modélisation et l'optimisation dans la Partie 2. Vous savez concevoir des schémas performants et construire des requêtes efficaces. Mais une question cruciale se pose maintenant : **comment garantir la cohérence des données dans un système distribué lorsque plusieurs opérations doivent réussir ou échouer ensemble ?**

La Partie 3 aborde l'un des sujets les plus complexes et les plus critiques de MongoDB : **les transactions et la concurrence**. C'est ici que se rejoignent la théorie des systèmes distribués et les besoins pragmatiques des applications critiques en production.

## 🌐 Le défi de la cohérence distribuée

### Le contexte : MongoDB comme système distribué

MongoDB, dès qu'on utilise un Replica Set (et encore plus avec le sharding), est un **système distribué**. Cela implique des défis fondamentaux :

- **La latence réseau** : Les données peuvent être répliquées sur plusieurs nœuds géographiquement distants
- **Les pannes partielles** : Un nœud peut échouer sans que les autres soient affectés
- **La synchronisation** : Les écritures doivent être propagées aux réplicas
- **La cohérence** : Comment garantir que tous les nœuds voient les mêmes données ?

### Le théorème CAP : Les compromis inévitables

Le théorème CAP, fondamental en systèmes distribués, stipule qu'un système ne peut garantir simultanément que **deux des trois propriétés** suivantes :

- **C** (Consistency) : Tous les nœuds voient les mêmes données au même moment
- **A** (Availability) : Le système répond toujours, même en cas de panne partielle
- **P** (Partition tolerance) : Le système continue de fonctionner malgré des coupures réseau

**Le positionnement de MongoDB :**

MongoDB privilégie **CP** (Cohérence + Tolérance au partitionnement), mais offre une **flexibilité configurable** :
- Vous pouvez choisir le niveau de cohérence (Read Concern)
- Vous pouvez choisir les garanties d'écriture (Write Concern)
- Vous pouvez sacrifier la cohérence forte pour la performance si votre cas d'usage le permet

> **Principe fondamental** : Dans MongoDB, vous ne subissez pas les compromis CAP, vous les **choisissez** en fonction de vos besoins.

### ACID dans un contexte NoSQL : Une évolution majeure

Historiquement, les bases NoSQL sacrifiaient les garanties ACID pour la scalabilité. MongoDB a changé la donne :

**ACID rappel :**
- **A** (Atomicity) : Une transaction s'exécute complètement ou pas du tout
- **C** (Consistency) : La base reste dans un état cohérent avant et après la transaction
- **I** (Isolation) : Les transactions concurrentes ne s'interfèrent pas
- **D** (Durability) : Une fois validée, une transaction est permanente

**L'évolution de MongoDB :**
- **Avant 3.x** : ACID uniquement au niveau mono-document
- **MongoDB 4.0 (2018)** : Transactions multi-documents sur Replica Sets
- **MongoDB 4.2 (2019)** : Transactions multi-documents sur clusters shardés
- **MongoDB 5.0+** : Optimisations et nouvelles garanties de cohérence

Cette évolution a transformé MongoDB d'une base NoSQL "eventually consistent" en une base capable de rivaliser avec les SGBDR pour les applications critiques.

## 🎯 Pourquoi cette partie est critique

### Les applications qui nécessitent des transactions

Certaines applications ne peuvent se permettre d'incohérences, même temporaires :

**1. Applications financières**
```
Exemple : Transfert bancaire
- Débiter le compte A : -100€
- Créditer le compte B : +100€
→ Les deux opérations doivent réussir ou échouer ensemble
```

**2. E-commerce**
```
Exemple : Commande de produit
- Réduire le stock : quantity -= 1
- Créer la commande
- Débiter le paiement
→ Toutes les opérations doivent être atomiques
```

**3. Gestion de réservations**
```
Exemple : Réservation de siège
- Marquer le siège comme réservé
- Créer la réservation utilisateur
- Enregistrer le paiement
→ Éviter les sur-réservations
```

**4. Systèmes d'inventaire**
```
Exemple : Gestion multi-entrepôts
- Allouer depuis l'entrepôt A
- Si insuffisant, essayer l'entrepôt B
- Mettre à jour les stocks
→ Cohérence entre entrepôts cruciale
```

### Le coût de l'incohérence

Sans transactions appropriées, vous risquez :
- ❌ **Perte de données** : Une opération réussit, l'autre échoue
- ❌ **Incohérences** : États invalides (compte négatif, stock négatif)
- ❌ **Problèmes légaux** : Non-conformité réglementaire (finance, santé)
- ❌ **Perte de confiance** : Clients affectés par des bugs de cohérence
- ❌ **Coûts de correction** : Réconciliation manuelle coûteuse

**Impact business réel :**
- Une banque avec des transactions mal gérées peut perdre des millions
- Un e-commerce avec des stocks incohérents génère des clients mécontents
- Une plateforme de réservation avec des doublons perd sa crédibilité

## 📋 Prérequis

Cette partie s'adresse à des développeurs et architectes **expérimentés** ayant :

### Connaissances techniques requises
- ✅ **Maîtrise complète de la Partie 1** : CRUD, requêtes, structures de données
- ✅ **Maîtrise complète de la Partie 2** : Modélisation, index, agrégation
- ✅ **Compréhension des systèmes distribués** : Réplication, cohérence, latence
- ✅ **Expérience en production** : Idéalement, avoir déployé une application MongoDB
- ✅ **Connaissance des Replica Sets** : Architecture, élection, oplog (abordé en Partie 4, mais un aperçu est utile)

### Prérequis conceptuels
- 🧠 Compréhension du théorème CAP
- 🧠 Notions de cohérence (forte vs éventuelle)
- 🧠 Compréhension des transactions en SQL (si applicable)
- 🧠 Connaissance des problèmes de concurrence (race conditions, deadlocks)

### État d'esprit
- 🎯 Capacité à analyser les compromis (trade-offs)
- 🎯 Rigueur dans la conception de systèmes critiques
- 🎯 Compréhension que la "meilleure" solution dépend du contexte
- 🎯 Volonté d'expérimenter et de mesurer les impacts

**Note importante :** Si vous n'êtes pas à l'aise avec les Replica Sets, consultez la Partie 4 (Réplication) en parallèle, car les transactions s'appuient sur l'architecture Replica Set.

## 🎓 Objectifs d'apprentissage

À la fin de cette partie, vous serez capable de :

### Compréhension théorique
- ✅ **Expliquer** le positionnement de MongoDB dans le théorème CAP
- ✅ **Différencier** cohérence forte, cohérence éventuelle, et isolation
- ✅ **Comprendre** les propriétés ACID dans un contexte distribué
- ✅ **Analyser** les compromis entre performance et cohérence
- ✅ **Justifier** l'utilisation de transactions vs atomicité native

### Compétences en transactions mono-document
- ✅ **Exploiter** l'atomicité native de MongoDB pour des opérations simples
- ✅ **Utiliser** les opérateurs atomiques ($inc, $push, $set, etc.)
- ✅ **Concevoir** des modèles qui minimisent le besoin de transactions multi-documents
- ✅ **Comprendre** les limites et avantages de l'atomicité mono-document

### Compétences en transactions multi-documents
- ✅ **Implémenter** des transactions multi-documents avec l'API Sessions
- ✅ **Gérer** correctement les commits et rollbacks
- ✅ **Gérer** les erreurs transactionnelles (retry logic)
- ✅ **Comprendre** les contraintes (limites de temps, taille, etc.)
- ✅ **Optimiser** les transactions pour la performance

### Compétences en cohérence et isolation
- ✅ **Configurer** les Read Concerns (local, available, majority, linearizable, snapshot)
- ✅ **Configurer** les Write Concerns (w, j, wtimeout)
- ✅ **Choisir** le bon niveau de cohérence selon le cas d'usage
- ✅ **Comprendre** les implications de performance de chaque niveau
- ✅ **Gérer** les transactions distribuées sur clusters shardés

### Compétences opérationnelles
- ✅ **Diagnostiquer** les problèmes de concurrence
- ✅ **Monitorer** les transactions en production
- ✅ **Optimiser** les performances transactionnelles
- ✅ **Appliquer** les bonnes pratiques transactionnelles
- ✅ **Gérer** les situations de contention et deadlocks

## 📚 Vue d'ensemble du module

Cette partie contient **un seul module dense** mais crucial :

### Module 8 : Transactions
**Durée estimée : 12-16 heures**

Ce module couvre tous les aspects des transactions dans MongoDB, de la théorie à la pratique avancée.

#### 8.1 Rappel ACID et Contexte NoSQL
**Durée : 2-3 heures**

Révision approfondie des propriétés ACID et de leur interprétation dans un système distribué NoSQL.

**Ce que vous maîtriserez :**
- Les définitions rigoureuses de l'Atomicité, Cohérence, Isolation et Durabilité
- ACID dans le contexte NoSQL vs SGBDR traditionnels
- La comparaison avec les bases relationnelles (Postgres, MySQL)
- Les défis spécifiques des systèmes distribués

**Pourquoi c'est important :** Comprendre la théorie vous permet de faire les bons choix pratiques. Beaucoup de bugs de production viennent d'une mauvaise compréhension de ces concepts.

---

#### 8.2 Atomicité Native : Transactions Mono-document
**Durée : 2-3 heures**

MongoDB offre une atomicité native au niveau du document, souvent suffisante et beaucoup plus performante que les transactions multi-documents.

**Ce que vous maîtriserez :**
- L'atomicité garantie par MongoDB pour les opérations mono-document
- Les opérateurs atomiques ($inc, $mul, $push, $pull, $addToSet, etc.)
- La conception de modèles qui maximisent l'utilisation de l'atomicité native
- Les cas où l'atomicité mono-document suffit

**Principe clé :** Si vous pouvez modéliser vos données pour que les opérations critiques touchent un seul document, vous n'avez pas besoin de transactions multi-documents. C'est le plus performant.

**Exemple :**
```javascript
// Au lieu de deux documents séparés (nécessitant une transaction) :
// users: { _id, name, balance }
// transactions: { _id, userId, amount }

// Imbriquer les transactions dans l'utilisateur (atomicité native) :
{
  _id: "user123",
  name: "Alice",
  balance: 1000,
  transactions: [
    { date: ISODate(), amount: -50, type: "debit" },
    { date: ISODate(), amount: 100, type: "credit" }
  ]
}

// Opération atomique :
db.users.updateOne(
  { _id: "user123", balance: { $gte: 50 } },
  { 
    $inc: { balance: -50 },
    $push: { transactions: { date: new Date(), amount: -50, type: "debit" } }
  }
)
```

---

#### 8.3 Transactions Multi-documents
**Durée : 4-6 heures**

Le cœur du module : les transactions qui touchent plusieurs documents ou collections.

**Ce que vous maîtriserez :**
- Les cas d'usage qui nécessitent vraiment des transactions multi-documents
- L'API des Sessions et Transactions
- La syntaxe complète (startSession, startTransaction, commitTransaction, abortTransaction)
- La gestion des erreurs et la retry logic
- Les limites techniques (60 secondes par défaut, 16 Mo de données modifiées)

**Syntaxe de base :**
```javascript
const session = client.startSession();
try {
  session.startTransaction();
  
  // Opérations transactionnelles
  await collection1.updateOne({ _id: 1 }, { $inc: { value: -100 } }, { session });
  await collection2.updateOne({ _id: 2 }, { $inc: { value: 100 } }, { session });
  
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  await session.endSession();
}
```

**Pourquoi c'est complexe :**
- Les transactions multi-documents ont un coût de performance significatif
- Elles nécessitent une gestion rigoureuse des erreurs
- Elles peuvent échouer pour des raisons réseau ou de contention
- Elles doivent être idempotentes pour permettre les retries

---

#### 8.4 Niveaux de Cohérence et d'Isolation
**Durée : 4-5 heures**

La partie la plus subtile : configurer les niveaux de cohérence appropriés.

**Ce que vous maîtriserez :**

**Read Concern** (ce que vous lisez) :
- `local` : Lecture depuis le nœud local (performant, peut lire des données non durables)
- `available` : Similaire à local (utilisé pour le sharding)
- `majority` : Lecture de données reconnues par la majorité des nœuds (durable, mais plus lent)
- `linearizable` : Garantie de cohérence linéarisable (le plus strict, le plus lent)
- `snapshot` : Lecture cohérente dans le temps (pour les transactions)

**Write Concern** (confirmation d'écriture) :
- `w: 1` : Écriture confirmée par le primaire uniquement (rapide, risque de perte)
- `w: "majority"` : Écriture confirmée par la majorité des nœuds (durable, plus lent)
- `j: true` : Écriture dans le journal (durable même en cas de crash)
- `wtimeout` : Timeout si la réplication prend trop de temps

**Compromis performance vs cohérence :**
```
                    Performance
                        ↑
local/w:1               |
                        |
majority/w:majority     |
                        |
linearizable/snapshot   |
                        ↓
                    Cohérence forte
```

**Règle d'or :** Utilisez le niveau de cohérence **minimum nécessaire** pour votre cas d'usage. Plus de cohérence = moins de performance.

---

#### 8.5 Transactions Distribuées
**Durée : 2-3 heures**

Les transactions sur clusters shardés : encore plus complexe.

**Ce que vous maîtriserez :**
- Les défis spécifiques du sharding
- Le protocole de commit en deux phases (2PC)
- Les implications de performance
- Les cas d'usage et les alternatives

**Réalité :** Les transactions distribuées sur shards sont coûteuses. Privilégiez :
1. La modélisation pour éviter les transactions cross-shard
2. L'utilisation de shard keys appropriées
3. Les transactions multi-documents sur un seul shard quand possible

---

#### 8.6 Limites et Considérations de Performance
**Durée : 1-2 heures**

Comprendre les contraintes et les optimiser.

**Limites techniques :**
- Timeout par défaut : 60 secondes
- Taille maximale des modifications : 16 Mo
- Pas plus de 1000 opérations dans une transaction
- Possible contention sur les documents très sollicités

**Considérations de performance :**
- Les transactions sont 10-100x plus lentes que les opérations atomiques
- Elles génèrent du overhead de synchronisation
- Elles peuvent causer des deadlocks si mal utilisées

**Stratégies d'optimisation :**
- Minimiser le nombre d'opérations par transaction
- Grouper les opérations sur les mêmes collections
- Utiliser des index appropriés
- Implémenter une retry logic intelligente
- Monitorer les transactions lentes

---

#### 8.7 Bonnes Pratiques Transactionnelles
**Durée : 1-2 heures**

Les patterns et anti-patterns pour les transactions en production.

**Bonnes pratiques :**
- ✅ Modéliser pour minimiser les transactions multi-documents
- ✅ Garder les transactions courtes (< 1 seconde idéalement)
- ✅ Implémenter une retry logic avec backoff exponentiel
- ✅ Utiliser des timeout appropriés
- ✅ Monitorer les transactions en production
- ✅ Tester les cas de contention et de timeout

**Anti-patterns à éviter :**
- ❌ Transactions longues (> 60 secondes)
- ❌ Trop d'opérations dans une seule transaction
- ❌ Transactions sans gestion d'erreurs
- ❌ Absence de retry logic
- ❌ Utilisation systématique de transactions quand l'atomicité native suffit
- ❌ Transactions imbriquées (non supporté)

## 🔄 Progression pédagogique

Cette partie suit une logique **théorie → pratique → optimisation** :

```
Comprendre ACID → Atomicité native → Transactions multi-docs → Cohérence → Optimisation
```

### Semaine 1 : Fondements théoriques et atomicité native
**Focus : Comprendre avant d'implémenter**

**Jours 1-2 : ACID et contexte**
- Révision approfondie des propriétés ACID
- Comparaison NoSQL vs SQL
- Théorème CAP et positionnement de MongoDB

**Jours 3-5 : Atomicité mono-document**
- Opérateurs atomiques ($inc, $push, etc.)
- Modélisation pour maximiser l'atomicité native
- Exercices pratiques

**Jours 6-7 : Révision et pratique**
- Cas d'usage réels
- Refactoring de modèles pour éviter les transactions

**Livrables :** Refactorer 3 modèles de données pour utiliser l'atomicité native

---

### Semaine 2 : Transactions multi-documents
**Focus : Maîtriser l'API et la gestion d'erreurs**

**Jours 1-3 : API et syntaxe**
- Sessions et transactions
- Commit et rollback
- Gestion d'erreurs basique

**Jours 4-5 : Scénarios avancés**
- Retry logic
- Gestion des timeouts
- Transactions concurrentes

**Jours 6-7 : Pratique intensive**
- Implémentation de cas d'usage réels
- Tests de robustesse

**Livrables :** Implémenter 3 workflows transactionnels (transfert, commande, réservation)

---

### Semaine 3 : Cohérence et optimisation
**Focus : Choisir les bons compromis**

**Jours 1-3 : Read et Write Concerns**
- Configuration et implications
- Benchmarking des différents niveaux
- Choix selon les cas d'usage

**Jours 4-5 : Transactions distribuées**
- Sharding et transactions
- Optimisations

**Jours 6-7 : Bonnes pratiques et monitoring**
- Patterns de production
- Monitoring et alerting
- Tests de charge

**Livrables :** Document de conception précisant les niveaux de cohérence pour chaque collection

---

**Rythme recommandé :** 3-4 heures par jour, avec des sessions intensives pour la pratique.

## 🧠 Concepts clés à maîtriser

### 1. L'atomicité mono-document est votre première option

**Principe :** 95% du temps, vous n'avez pas besoin de transactions multi-documents si votre modélisation est bonne.

**Application :**
```javascript
// ❌ Mauvais : Nécessite une transaction
// Collection users: { _id, balance }
// Collection orders: { _id, userId, total }

// ✅ Bon : Atomicité native
// Collection users: { 
//   _id, 
//   balance, 
//   orders: [{ total, date, items }] 
// }
```

### 2. Les transactions ont un coût

**Overhead typique :**
- Latence : +20-50ms par transaction
- Throughput : -30-50% par rapport à l'atomicité native
- Ressources : Plus de CPU, RAM, réseau

**Conséquence :** Utilisez-les uniquement quand nécessaire.

### 3. La cohérence est un spectre, pas un booléen

**Faux :** "Mes données sont cohérentes ou non"
**Vrai :** "J'ai choisi le niveau de cohérence adapté à chaque opération"

**Exemple :**
- Solde bancaire : `majority` + `w: "majority"` (cohérence forte requise)
- Compteur de vues : `local` + `w: 1` (cohérence éventuelle acceptable)
- Analytics : `available` (disponibilité > cohérence)

### 4. L'idempotence est essentielle

**Pourquoi :** Les transactions peuvent échouer et être rejouées (retry logic).

**Comment :**
```javascript
// ❌ Non idempotent
db.accounts.updateOne(
  { _id: "acc1" },
  { $inc: { balance: 100 } }
)

// ✅ Idempotent (avec identifiant de transaction)
db.accounts.updateOne(
  { _id: "acc1", "transactions.txId": { $ne: "tx123" } },
  { 
    $inc: { balance: 100 },
    $push: { transactions: { txId: "tx123", amount: 100 } }
  }
)
```

### 5. Monitorer, toujours monitorer

**Métriques critiques :**
- Durée moyenne des transactions
- Taux de commit vs abort
- Contention sur les documents
- Transactions en cours (currentOp)

**Alertes à configurer :**
- Transactions > 10 secondes
- Taux d'échec > 5%
- Contention élevée

## 🚦 Validation des acquis

Avant de passer à la Partie 4, vous devez maîtriser :

### Checklist théorique
- [ ] Je peux expliquer le théorème CAP et le positionnement de MongoDB
- [ ] Je comprends les propriétés ACID dans un contexte distribué
- [ ] Je peux différencier cohérence forte, éventuelle, et linearizable
- [ ] Je comprends les compromis entre performance et cohérence

### Checklist pratique
- [ ] Je sais quand utiliser l'atomicité native vs transactions multi-documents
- [ ] Je peux implémenter une transaction multi-documents avec gestion d'erreurs
- [ ] Je sais configurer Read Concern et Write Concern appropriés
- [ ] Je peux diagnostiquer et résoudre des problèmes de concurrence
- [ ] J'ai implémenté une retry logic robuste

### Checklist opérationnelle
- [ ] Je peux monitorer les transactions en production
- [ ] Je connais les limites techniques et comment les gérer
- [ ] J'ai testé mes transactions sous charge
- [ ] Je peux justifier mes choix de cohérence dans un design doc

### Checklist architecturale
- [ ] Je modélise d'abord pour éviter les transactions
- [ ] Je connais au moins 3 patterns pour minimiser les transactions
- [ ] Je peux concevoir un système critique avec garanties ACID
- [ ] Je comprends l'impact des transactions sur le sharding

**Objectif :** Cocher 90%+ de ces cases avant de continuer.

## 🎯 Projet pratique recommandé

### Projet : Système de transfert inter-comptes avec historique

**Contexte :** Implémentez un système financier simplifié avec garanties ACID.

**Fonctionnalités :**
1. Création de comptes avec solde initial
2. Transfert d'argent entre comptes (atomique)
3. Historique des transactions
4. Vérification de cohérence (soldes ne peuvent être négatifs)
5. Support de transactions concurrentes
6. Retry logic en cas d'échec

**Exigences techniques :**
- Modéliser avec atomicité native où possible
- Implémenter les transferts avec transactions multi-documents
- Utiliser `majority` pour les Read/Write Concerns
- Gérer tous les cas d'erreur (insuffisant de fonds, timeout, etc.)
- Implémenter une retry logic avec backoff exponentiel
- Monitorer les transactions (durée, succès/échec)
- Tester sous charge avec 100+ transactions/seconde
- Documenter les choix de cohérence

**Cas de test critiques :**
- Transfert simple qui réussit
- Transfert avec solde insuffisant (doit échouer)
- Transferts concurrents sur le même compte
- Panne réseau pendant la transaction
- Timeout de transaction

**Livrables :**
- Code complet avec tests
- Document de conception expliquant les choix ACID
- Benchmarks de performance (avec et sans transactions)
- Rapport de tests de charge
- Plan de monitoring pour la production

**Durée estimée :** 20-25 heures

**Critères de validation :**
- ✅ Aucune perte d'argent dans le système (somme des soldes constante)
- ✅ Aucun solde négatif
- ✅ Historique complet et cohérent
- ✅ Support de 100+ TPS sans corruption
- ✅ Retry automatique en cas d'échec transitoire

Ce projet vous donnera une expérience pratique complète des transactions MongoDB en conditions réelles.

## 📊 Comparaison : Atomicité native vs Transactions multi-documents

| Critère | Atomicité Native | Transactions Multi-docs |
|---------|------------------|-------------------------|
| **Performance** | ⚡️ Très rapide (< 1ms) | 🐌 Lent (20-50ms+) |
| **Complexité** | ✅ Simple | ⚠️ Complexe (gestion erreurs, retry) |
| **Scalabilité** | ⚡️ Excellente | ⚠️ Limitée (contention possible) |
| **Cas d'usage** | 90% des besoins | 10% des cas critiques |
| **Overhead** | Minimal | Significatif (CPU, RAM, réseau) |
| **Garanties ACID** | ✅ Sur 1 document | ✅ Sur N documents |
| **Risques** | ❌ Limité à 1 document | ⚠️ Deadlocks, timeouts possibles |

**Conclusion :** Privilégiez toujours l'atomicité native. N'utilisez les transactions multi-documents que quand c'est absolument nécessaire.

## 🌟 Conseils avancés

### 1. Concevez pour éviter les transactions
La meilleure transaction est celle que vous n'avez pas besoin d'écrire. Investissez dans la modélisation.

### 2. Testez sous charge dès le début
Les problèmes de concurrence n'apparaissent que sous charge. Testez avec des outils comme `mongodb-loadtest` ou `artillery`.

### 3. Implémentez une observabilité complète
Loggez chaque transaction : durée, opérations, résultat. Vous en aurez besoin pour le debug.

### 4. Prévoyez le pire
Vos transactions vont échouer. Planifiez la retry logic, les compensations, et les alertes dès la conception.

### 5. Documentez vos choix de cohérence
Dans 6 mois, vous (ou votre équipe) devrez justifier pourquoi telle collection utilise `majority` et telle autre `local`. Documentez maintenant.

### 6. Benchmarkez tout
Ne supposez pas. Mesurez l'impact réel des transactions sur votre workload spécifique.

## 📚 Ressources complémentaires

### Documentation officielle
- [Transactions in MongoDB](https://www.mongodb.com/docs/manual/core/transactions/)
- [Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/)
- [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [Production Considerations](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/)

### Lectures avancées
- *Designing Data-Intensive Applications* par Martin Kleppmann (Chapitre sur les transactions)
- Papiers académiques sur le 2PC (Two-Phase Commit)
- Articles MongoDB Engineering Blog sur les transactions

### Outils
- MongoDB Ops Manager (monitoring transactionnel)
- MongoDB Atlas (métriques de transactions)
- Profiler MongoDB (analyse des transactions lentes)

## 🚀 Et après ?

Une fois cette partie maîtrisée, vous comprendrez **comment garantir la cohérence des données** même dans des systèmes distribués complexes. Vous saurez :

- Quand utiliser les transactions et quand les éviter
- Comment configurer les niveaux de cohérence appropriés
- Comment implémenter des systèmes critiques avec garanties ACID
- Comment optimiser les performances transactionnelles

La **Partie 4** vous enseignera l'architecture distribuée (Réplication et Sharding), vous permettant de comprendre le fonctionnement interne de MongoDB et de déployer des systèmes hautement disponibles et scalables.

La **Partie 5** abordera la sécurité et l'administration, essentielles pour mettre en production des systèmes utilisant des transactions.

Mais d'abord, **maîtrisez cette Partie 3**. Les transactions sont subtiles. Une mauvaise compréhension peut mener à des bugs de production critiques. Prenez le temps de vraiment comprendre les concepts et de pratiquer intensivement.

---

**Prêt à garantir la cohérence de vos données ? Allons-y ! 🔒**

---

**Prochaine étape :** [Module 8 - Transactions →](/08-transactions/README.md)

---

*💡 Citation du jour : "In theory, there is no difference between theory and practice. In practice, there is." - Yogi Berra (parfaitement applicable aux transactions distribuées)*

⏭️ [Module 8 - Transactions →](/08-transactions/README.md)
