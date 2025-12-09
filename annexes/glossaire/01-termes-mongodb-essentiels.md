🔝 Retour au [Sommaire](/SOMMAIRE.md)

# A.1 - Termes MongoDB Essentiels

## Table des matières

1. [Concepts Fondamentaux](#concepts-fondamentaux)
2. [Architecture Distribuée](#architecture-distribu%C3%A9e)
3. [Index et Optimisation](#index-et-optimisation)
4. [Transactions et Cohérence](#transactions-et-coh%C3%A9rence)
5. [Opérations et Commandes](#op%C3%A9rations-et-commandes)
6. [Sécurité](#s%C3%A9curit%C3%A9)
7. [Outils et Composants](#outils-et-composants)
8. [Monitoring et Métriques](#monitoring-et-m%C3%A9triques)

---

## Concepts Fondamentaux

### BSON **[Débutant]**
🔤 **Binary JSON** - Format de sérialisation binaire utilisé par MongoDB pour stocker et échanger des documents. Extension de JSON supportant des types additionnels (Date, ObjectId, Binary, etc.).

💡 **Cas d'usage** : Format natif de stockage dans MongoDB, plus compact et plus rapide à parser que JSON.

📌 **Exemple** : Un document JSON `{"name": "Alice"}` est stocké en BSON avec métadonnées de type.

⚠️ **Limitation** : Taille maximale d'un document BSON = 16 Mo

🔗 **Voir aussi** : Document, ObjectId, Types de données

---

### Document **[Débutant]**
🔤 Unité de base de stockage dans MongoDB, équivalent d'une ligne/enregistrement en SQL. Structure de données au format BSON contenant des paires clé-valeur.

💡 **Cas d'usage** : Représentation flexible de données complexes, imbrication possible (sous-documents, tableaux).

📌 **Exemple** :
```json
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "prenom": "Jean",
  "adresse": {
    "rue": "5 rue de la Paix",
    "ville": "Paris"
  }
}
```

🎯 **Bonne pratique** : Modéliser selon les patterns d'accès, pas selon la normalisation relationnelle.

---

### Collection **[Débutant]**
🔤 Groupe de documents MongoDB, équivalent d'une table en SQL. Ne impose pas de schéma fixe (schema-less).

💡 **Cas d'usage** : Organiser des documents de même nature (ex: collection "utilisateurs", "commandes").

⚠️ **À noter** : Absence de schéma obligatoire ne signifie pas absence de structure recommandée. Utiliser la validation de schéma en production.

🔗 **Voir aussi** : Database, Document, Validation de schéma

---

### Database **[Débutant]**
🔤 Conteneur logique de collections. Unité d'isolation physique (fichiers distincts sur disque).

💡 **Cas d'usage** : Séparer les environnements (dev, test, prod) ou les applications.

📌 **Exemple** : `use maBaseDeDonnees` pour basculer sur une base.

🎯 **Bonne pratique** : Une base par application ou domaine métier.

---

### ObjectId **[Débutant]**
🔤 Identifiant unique de 12 octets généré automatiquement pour chaque document (champ `_id` par défaut).

💡 **Composition** :
- 4 octets : timestamp de création
- 5 octets : valeur aléatoire (process ID + compteur machine)
- 3 octets : compteur incrémental

📌 **Exemple** : `ObjectId("507f1f77bcf86cd799439011")`

💡 **Avantage** : Génération décentralisée (pas de séquence globale), tri chronologique naturel.

🔗 **Voir aussi** : _id, Index, Sharding

---

### _id **[Débutant]**
🔤 Champ identifiant unique obligatoire dans chaque document. Index automatique créé sur ce champ.

⚠️ **Important** : Immuable après insertion. Si non fourni, MongoDB génère un ObjectId.

💡 **Types possibles** : ObjectId (défaut), String, Number, UUID, etc.

🎯 **Bonne pratique** : Utiliser ObjectId sauf besoin métier spécifique (identifiants externes).

---

### Schéma **[Débutant]**
🔤 Structure des données dans un document. MongoDB est "schema-flexible" (pas de schéma rigide imposé).

💡 **Schema-less vs Schema design** :
- Pas de schéma = liberté de structure par document
- Mais recommandé de définir un schéma implicite cohérent

🎯 **Bonne pratique** : Utiliser la validation de schéma ($jsonSchema) en production pour garantir la qualité des données.

🔗 **Voir aussi** : Validation de schéma, Patterns de modélisation

---

### Namespace **[Intermédiaire]**
🔤 Combinaison database.collection formant un identifiant complet d'une collection.

📌 **Exemple** : `myapp.users` = collection "users" dans la base "myapp"

💡 **Usage** : Utilisé en interne par MongoDB pour les logs, les opérations, etc.

---

## Architecture Distribuée

### Replica Set **[Intermédiaire]**
🔤 Groupe de serveurs MongoDB maintenant les mêmes données pour assurer haute disponibilité et redondance.

💡 **Composition** :
- 1 Primary (accepte les écritures)
- N Secondaries (répliquent les données)
- Optionnel : Arbiters (vote uniquement, pas de données)

📌 **Minimum recommandé** : 3 nœuds (1 Primary + 2 Secondaries)

🎯 **Avantage** : Tolérance aux pannes, lecture distribuée possible.

⚠️ **À noter** : Élection automatique d'un nouveau Primary en cas de panne.

🔗 **Voir aussi** : Primary, Secondary, Arbiter, Oplog, Election, Failover

---

### Primary **[Intermédiaire]**
🔤 Membre d'un Replica Set qui reçoit toutes les opérations d'écriture. Un seul Primary actif à la fois.

💡 **Rôle** : Point d'entrée unique pour les écritures, propage les changements aux Secondaries via l'Oplog.

🔗 **Voir aussi** : Replica Set, Secondary, Election

---

### Secondary **[Intermédiaire]**
🔤 Membre d'un Replica Set qui réplique les données du Primary. Peut servir les lectures (selon Read Preference).

💡 **Rôle** : Redondance, haute disponibilité, distribution de charge en lecture.

⚠️ **À noter** : Données éventuellement en retard (replication lag).

🔗 **Voir aussi** : Replica Set, Primary, Read Preference

---

### Arbiter **[Intermédiaire]**
🔤 Membre d'un Replica Set qui participe uniquement aux élections (vote), sans stocker de données.

💡 **Cas d'usage** : Obtenir un nombre impair de votants sans coût de stockage supplémentaire.

⚠️ **Limitation** : Ne peut pas devenir Primary, ne sert pas les lectures.

🎯 **Recommandation** : Préférer un vrai Secondary si possible (meilleure résilience).

---

### Oplog **[Intermédiaire]**
🔤 **Operations Log** - Collection cappée spéciale (`local.oplog.rs`) contenant l'historique des opérations d'écriture sur le Primary.

💡 **Rôle** : Réplication des données vers les Secondaries, base du mécanisme de réplication.

📌 **Caractéristiques** :
- Taille fixe (configurable)
- FIFO (First In First Out)
- Format idempotent (rejouer une opération = même résultat)

⚠️ **Dimensionnement** : Doit contenir assez d'historique pour permettre la resynchronisation d'un Secondary temporairement déconnecté.

🔗 **Voir aussi** : Replica Set, Capped Collection, Replication Lag

---

### Élection **[Intermédiaire]**
🔤 Processus automatique de sélection d'un nouveau Primary lors d'une panne ou maintenance.

💡 **Mécanisme** : Vote majoritaire basé sur la priorité et la fraîcheur des données.

⚠️ **Durée** : Généralement 10-30 secondes (période d'indisponibilité en écriture).

🔗 **Voir aussi** : Replica Set, Primary, Priority

---

### Failover **[Intermédiaire]**
🔤 Basculement automatique vers un nouveau Primary en cas de défaillance du Primary actuel.

💡 **Avantage** : Haute disponibilité sans intervention manuelle.

⚠️ **Impact** : Brève période d'indisponibilité en écriture pendant l'élection.

🔗 **Voir aussi** : Replica Set, Élection, Haute disponibilité

---

### Sharding **[Avancé]**
🔤 Méthode de distribution horizontale des données sur plusieurs serveurs (shards) pour gérer de très gros volumes.

💡 **Objectif** : Scalabilité horizontale, distribution de charge.

📌 **Composants** :
- Shards : stockent les données
- Config Servers : métadonnées du cluster
- Mongos : routeurs de requêtes

⚠️ **Complexité** : Architecture plus complexe, choix critique de la Shard Key.

🔗 **Voir aussi** : Shard, Shard Key, Chunk, Mongos, Config Servers

---

### Shard **[Avancé]**
🔤 Serveur (ou Replica Set) dans un cluster shardé qui stocke un sous-ensemble des données.

💡 **Distribution** : Chaque shard contient des chunks différents selon la Shard Key.

🔗 **Voir aussi** : Sharding, Chunk, Replica Set

---

### Shard Key **[Avancé]**
🔤 Champ(s) indexé(s) utilisé(s) pour distribuer les documents entre les shards.

💡 **Critères de choix** :
- Cardinalité élevée (nombreuses valeurs distinctes)
- Distribution uniforme
- Éviter les hot spots (concentration sur un shard)

⚠️ **Critique** : Choix difficilement réversible, impact majeur sur les performances.

📌 **Exemples** :
- Bon : `user_id` (UUID aléatoire)
- Mauvais : `timestamp` (séquentiel, hot spot)

🔗 **Voir aussi** : Sharding, Hashed Sharding, Range Sharding

---

### Chunk **[Avancé]**
🔤 Plage contiguë de données définies par la Shard Key, unité de distribution dans le sharding.

💡 **Taille par défaut** : 64 Mo (configurable)

💡 **Migration** : Les chunks sont automatiquement rééquilibrés entre shards par le Balancer.

⚠️ **Jumbo Chunk** : Chunk > 64 Mo qui ne peut être divisé, problème de performance potentiel.

🔗 **Voir aussi** : Sharding, Shard Key, Balancer

---

### Mongos **[Avancé]**
🔤 Processus routeur dans un cluster shardé qui dirige les requêtes vers les bons shards.

💡 **Rôle** : Point d'entrée pour les applications, masque la complexité du sharding.

🎯 **Déploiement** : Plusieurs instances mongos pour la haute disponibilité.

🔗 **Voir aussi** : Sharding, Config Servers

---

### Config Servers **[Avancé]**
🔤 Replica Set spécial stockant les métadonnées d'un cluster shardé (mapping chunks ↔ shards).

💡 **Nombre requis** : 3 membres minimum (Replica Set)

⚠️ **Critique** : Indispensables au fonctionnement du cluster, sauvegardes essentielles.

🔗 **Voir aussi** : Sharding, Mongos

---

### Balancer **[Avancé]**
🔤 Processus automatique qui migre les chunks entre shards pour maintenir une distribution équilibrée.

💡 **Fonctionnement** : Actif par défaut, peut être désactivé temporairement (maintenance).

⚠️ **Impact** : Consomme des ressources, planifier les migrations en heures creuses si besoin.

🔗 **Voir aussi** : Chunk, Sharding

---

## Index et Optimisation

### Index **[Débutant]**
🔤 Structure de données spéciale qui améliore la vitesse des requêtes en permettant un accès rapide aux documents.

💡 **Analogie** : Index d'un livre permettant de trouver rapidement une page.

⚠️ **Compromis** : Accélère les lectures, ralentit légèrement les écritures, consomme de l'espace disque.

📌 **Types principaux** : Single field, Compound, Multikey, Text, Geospatial, Hashed.

🎯 **Règle d'or** : Indexer les champs fréquemment utilisés dans les filtres, tris et jointures.

🔗 **Voir aussi** : Index composé, Multikey, Covered Query

---

### Index Composé (Compound Index) **[Intermédiaire]**
🔤 Index sur plusieurs champs, ordre des champs crucial pour l'efficacité.

💡 **Règle ESR** :
- **E**quality (=) en premier
- **S**ort en deuxième
- **R**ange (<, >, ≤, ≥) en dernier

📌 **Exemple** : Index `{status: 1, date: -1}` optimise `{status: "active"}` trié par date décroissante.

🔗 **Voir aussi** : Index, Query Planner

---

### Index Multikey **[Intermédiaire]**
🔤 Index automatiquement créé sur un champ tableau, indexant chaque élément du tableau.

💡 **Cas d'usage** : Requêtes sur des tableaux (tags, catégories, etc.).

⚠️ **Limitation** : Un seul champ tableau par index composé.

📌 **Exemple** : Index sur `tags` permet de chercher `{tags: "mongodb"}` efficacement.

---

### Index Unique **[Débutant]**
🔤 Index garantissant l'unicité des valeurs pour un champ ou une combinaison de champs.

💡 **Cas d'usage** : Email, numéro de sécurité sociale, identifiants métier.

⚠️ **Comportement** : Rejette les insertions/mises à jour créant des doublons.

📌 **Exemple** : `db.users.createIndex({email: 1}, {unique: true})`

---

### Index Sparse **[Intermédiaire]**
🔤 Index ne contenant que les documents possédant le champ indexé (ignore les documents sans ce champ).

💡 **Avantage** : Économie d'espace disque si le champ est optionnel.

⚠️ **Comportement** : Requêtes sur champs null ne pourront pas utiliser cet index.

🔗 **Voir aussi** : Index partiel

---

### Index Partiel (Partial Index) **[Intermédiaire]**
🔤 Index ne contenant que les documents satisfaisant un filtre spécifique.

💡 **Avantage** : Réduit la taille de l'index et améliore les performances pour des sous-ensembles de données.

📌 **Exemple** : Index uniquement sur les documents actifs :
```javascript
{status: 1}, {partialFilterExpression: {status: "active"}}
```

🔗 **Voir aussi** : Index sparse

---

### Index TTL (Time-To-Live) **[Intermédiaire]**
🔤 Index spécial qui supprime automatiquement les documents après un délai défini.

💡 **Cas d'usage** : Logs, sessions temporaires, caches.

📌 **Exemple** : Supprimer après 24h :
```javascript
db.logs.createIndex({createdAt: 1}, {expireAfterSeconds: 86400})
```

⚠️ **Fonctionnement** : Thread de nettoyage s'exécute toutes les 60 secondes.

---

### Index Texte (Text Index) **[Intermédiaire]**
🔤 Index spécialisé pour la recherche full-text (mots-clés, recherche textuelle).

💡 **Fonctionnalités** : Stemming, stop words, scores de pertinence.

⚠️ **Limitation** : Un seul index texte par collection.

📌 **Exemple** : `db.articles.createIndex({title: "text", content: "text"})`

🔗 **Voir aussi** : Atlas Search (solution plus avancée)

---

### Index Géospatial **[Avancé]**
🔤 Index pour requêtes géographiques (proximité, inclusion dans une zone).

💡 **Types** :
- **2d** : Plans euclidiens
- **2dsphere** : Sphère terrestre (coordonnées GPS)

📌 **Cas d'usage** : Trouver restaurants à proximité, livraisons dans une zone.

---

### Index Haché (Hashed Index) **[Avancé]**
🔤 Index où les valeurs sont hachées pour une distribution uniforme.

💡 **Cas d'usage** : Shard Key pour éviter les hot spots.

⚠️ **Limitation** : Ne supporte que l'égalité (pas les requêtes de plage).

🔗 **Voir aussi** : Hashed Sharding

---

### Covered Query **[Avancé]**
🔤 Requête entièrement satisfaite par un index, sans accès aux documents (ultra-rapide).

💡 **Conditions** :
- Tous les champs de la requête et projection sont dans l'index
- Pas de champ `_id` dans la projection (sauf s'il est dans l'index)

📌 **Exemple** :
```javascript
// Index : {name: 1, age: 1}
db.users.find({name: "Alice"}, {name: 1, age: 1, _id: 0})
```

🎯 **Performance** : 10-100x plus rapide qu'un scan de document.

---

### Query Planner **[Intermédiaire]**
🔤 Composant de MongoDB qui analyse une requête et choisit le meilleur plan d'exécution (quel index utiliser).

💡 **Fonctionnement** : Teste plusieurs plans en cache, sélectionne le plus performant.

📌 **Outil** : `explain()` pour voir le plan choisi.

🔗 **Voir aussi** : explain(), Index

---

## Transactions et Cohérence

### Transaction **[Intermédiaire]**
🔤 Ensemble d'opérations exécutées de manière atomique (tout ou rien).

💡 **Support** :
- Mono-document : natif depuis toujours
- Multi-documents : depuis MongoDB 4.0 (Replica Set) et 4.2 (Sharded)

📌 **Exemple** :
```javascript
const session = client.startSession();
session.startTransaction();
// ... opérations
session.commitTransaction();
```

⚠️ **Performance** : Plus coûteux que des opérations simples, à utiliser avec parcimonie.

🔗 **Voir aussi** : ACID, Session, Write Concern, Read Concern

---

### Session **[Intermédiaire]**
🔤 Contexte d'exécution pour grouper des opérations (requis pour les transactions multi-documents).

💡 **Usage** : Associer plusieurs opérations à une même transaction.

🔗 **Voir aussi** : Transaction

---

### Write Concern **[Intermédiaire]**
🔤 Niveau de confirmation requis pour une opération d'écriture.

💡 **Niveaux principaux** :
- `w: 1` : Acquittement du Primary uniquement (défaut)
- `w: "majority"` : Majorité des membres du Replica Set
- `j: true` : Écriture dans le journal (durabilité)

⚠️ **Compromis** : Plus de garantie = latence plus élevée.

🔗 **Voir aussi** : Read Concern, Transaction, Durabilité

---

### Read Concern **[Intermédiaire]**
🔤 Niveau de garantie sur la fraîcheur et la cohérence des données lues.

💡 **Niveaux principaux** :
- `local` : Données du nœud courant (défaut)
- `majority` : Données reconnues par la majorité
- `linearizable` : Garantie de linéarité stricte (plus lent)
- `snapshot` : Vue cohérente dans une transaction

🔗 **Voir aussi** : Write Concern, Transaction

---

### Consistency (Cohérence) **[Avancé]**
🔤 Garantie que les données respectent des règles définies.

💡 **Types dans MongoDB** :
- **Eventual Consistency** : Cohérence atteinte après un délai
- **Strong Consistency** : Cohérence immédiate (via Read Concern majority)

🔗 **Voir aussi** : CAP, Read Concern, Write Concern

---

### Atomicité **[Intermédiaire]**
🔤 Propriété garantissant qu'une opération est exécutée complètement ou pas du tout (pas d'état intermédiaire).

💡 **Dans MongoDB** :
- Atomicité native au niveau document
- Atomicité multi-documents via transactions

🔗 **Voir aussi** : ACID, Transaction

---

## Opérations et Commandes

### CRUD **[Débutant]**
🔤 **Create, Read, Update, Delete** - Quatre opérations de base sur les données.

💡 **Méthodes MongoDB** :
- Create : `insertOne()`, `insertMany()`
- Read : `find()`, `findOne()`
- Update : `updateOne()`, `updateMany()`, `replaceOne()`
- Delete : `deleteOne()`, `deleteMany()`

---

### Aggregation Pipeline **[Intermédiaire]**
🔤 Framework puissant pour le traitement de données via une séquence d'étapes (stages).

💡 **Étapes courantes** : `$match`, `$group`, `$project`, `$sort`, `$lookup`, `$unwind`

📌 **Exemple** :
```javascript
db.orders.aggregate([
  {$match: {status: "shipped"}},
  {$group: {_id: "$customerId", total: {$sum: "$amount"}}}
])
```

🎯 **Avantage** : Traitement côté serveur, optimisé, puissant.

🔗 **Voir aussi** : Pipeline, Stage, Operators

---

### Projection **[Débutant]**
🔤 Sélection des champs à retourner dans une requête (évite de transférer des données inutiles).

💡 **Syntaxe** :
- `1` : inclure le champ
- `0` : exclure le champ

📌 **Exemple** : `db.users.find({}, {name: 1, email: 1, _id: 0})`

⚠️ **Règle** : Impossible de mélanger inclusion et exclusion (sauf pour `_id`).

---

### Upsert **[Intermédiaire]**
🔤 Opération combinée : met à jour un document existant ou l'insère s'il n'existe pas.

💡 **Usage** : `updateOne({...}, {...}, {upsert: true})`

📌 **Cas d'usage** : Compteurs, métriques, caches.

---

### Bulk Operations **[Intermédiaire]**
🔤 Exécution groupée de plusieurs opérations pour améliorer les performances.

💡 **Types** :
- Ordered : arrêt à la première erreur
- Unordered : toutes les opérations exécutées, erreurs rapportées à la fin

📌 **Exemple** :
```javascript
db.collection.bulkWrite([
  {insertOne: {document: {...}}},
  {updateOne: {filter: {...}, update: {...}}}
])
```

---

## Sécurité

### Authentification **[Intermédiaire]**
🔤 Processus de vérification de l'identité d'un utilisateur ou application.

💡 **Mécanismes** :
- SCRAM (défaut)
- x.509 (certificats)
- LDAP
- Kerberos

🎯 **Recommandation** : Toujours activer en production.

🔗 **Voir aussi** : Autorisation, Utilisateur, Rôle

---

### Autorisation **[Intermédiaire]**
🔤 Contrôle des actions qu'un utilisateur authentifié peut effectuer (permissions).

💡 **Mécanisme** : Basé sur les rôles (RBAC - Role-Based Access Control).

🔗 **Voir aussi** : Rôle, Privilège, Authentification

---

### Rôle (Role) **[Intermédiaire]**
🔤 Ensemble de privilèges définissant ce qu'un utilisateur peut faire sur une ressource.

💡 **Rôles intégrés** : `read`, `readWrite`, `dbAdmin`, `userAdmin`, `clusterAdmin`, etc.

💡 **Rôles personnalisés** : Créés selon besoins métier spécifiques.

📌 **Exemple** : Attribuer le rôle readWrite :
```javascript
db.createUser({
  user: "appUser",
  pwd: "password",
  roles: [{role: "readWrite", db: "mydb"}]
})
```

🔗 **Voir aussi** : Autorisation, Privilège, Utilisateur

---

### Chiffrement **[Avancé]**
🔤 Protection des données par cryptographie.

💡 **Types dans MongoDB** :
- **En transit** : TLS/SSL pour les connexions réseau
- **Au repos** : Chiffrement des fichiers sur disque
- **CSFLE** : Chiffrement au niveau des champs (côté client)
- **Queryable Encryption** : Chiffrement interrogeable

🎯 **Production** : Chiffrement en transit obligatoire, chiffrement au repos recommandé.

---

### Audit **[Avancé]**
🔤 Enregistrement des activités pour la conformité et la sécurité.

💡 **Informations loggées** : Connexions, requêtes DDL, modifications de sécurité.

⚠️ **Disponibilité** : Édition Enterprise uniquement.

---

## Outils et Composants

### mongosh **[Débutant]**
🔤 **MongoDB Shell** - Interface en ligne de commande interactive pour MongoDB (remplace l'ancien mongo shell).

💡 **Capacités** : JavaScript moderne, syntaxe améliorée, auto-complétion.

📌 **Exemple** : `mongosh "mongodb://localhost:27017"`

---

### MongoDB Compass **[Débutant]**
🔤 Interface graphique officielle pour MongoDB (GUI).

💡 **Fonctionnalités** : Visualisation des données, création de requêtes, analyse de schéma, monitoring.

🎯 **Usage** : Idéal pour exploration, développement, débogage.

---

### MongoDB Atlas **[Intermédiaire]**
🔤 Service de base de données MongoDB entièrement géré dans le cloud (DBaaS).

💡 **Fournisseurs** : AWS, Azure, GCP

💡 **Avantages** : Déploiement simplifié, backups automatiques, monitoring intégré, scaling facile.

🔗 **Voir aussi** : Atlas Search, Atlas Data Lake, Atlas Triggers

---

### mongodump / mongorestore **[Intermédiaire]**
🔤 Outils de sauvegarde et restauration de données MongoDB.

💡 **Format** : BSON (binaire, compact)

📌 **Exemple** :
```bash
mongodump --db=mydb --out=/backup/
mongorestore --db=mydb /backup/mydb/
```

⚠️ **Limitation** : Ne préserve pas les indexes pendant le dump (recréés à la restauration).

---

### mongostat / mongotop **[Intermédiaire]**
🔤 Outils de monitoring en temps réel.

💡 **mongostat** : Statistiques globales (ops/sec, connexions, mémoire)
💡 **mongotop** : Temps passé en lecture/écriture par collection

📌 **Usage** : Diagnostic rapide de performance.

---

### WiredTiger **[Avancé]**
🔤 Moteur de stockage par défaut de MongoDB (depuis 3.2).

💡 **Caractéristiques** :
- Compression des données
- Concurrence au niveau document
- Checkpointing automatique
- Cache interne

🔗 **Voir aussi** : Storage Engine, Cache

---

### Storage Engine **[Avancé]**
🔤 Composant gérant le stockage physique des données sur disque.

💡 **Moteurs disponibles** :
- WiredTiger (défaut, recommandé)
- In-Memory (données volatiles)

---

## Monitoring et Métriques

### Profiler **[Intermédiaire]**
🔤 Outil de profilage enregistrant les opérations lentes dans une collection spéciale.

💡 **Niveaux** :
- 0 : désactivé
- 1 : requêtes lentes uniquement (> seuil)
- 2 : toutes les opérations

📌 **Activation** : `db.setProfilingLevel(1, 100)` (seuil 100ms)

⚠️ **Impact** : Niveau 2 dégrade les performances, usage temporaire uniquement.

---

### explain() **[Intermédiaire]**
🔤 Méthode retournant le plan d'exécution d'une requête.

💡 **Modes** :
- `queryPlanner` : Plan sélectionné
- `executionStats` : Statistiques d'exécution
- `allPlansExecution` : Tous les plans testés

📌 **Exemple** : `db.users.find({age: 25}).explain("executionStats")`

🎯 **Usage** : Diagnostic de performance, validation d'utilisation d'index.

---

### currentOp **[Intermédiaire]**
🔤 Commande affichant les opérations en cours d'exécution.

💡 **Usage** : Identifier les requêtes longues, blocages.

📌 **Exemple** : `db.currentOp({secs_running: {$gte: 10}})`

🔗 **Voir aussi** : killOp, Profiler

---

### Replication Lag **[Avancé]**
🔤 Retard de réplication entre Primary et Secondaries (en secondes).

💡 **Causes** : Charge élevée, réseau lent, opérations volumineuses.

⚠️ **Impact** : Lectures potentiellement obsolètes sur Secondaries.

🎯 **Monitoring** : Alertes si lag > seuil acceptable (ex: 30s).

---

### Working Set **[Avancé]**
🔤 Ensemble des données et index fréquemment accédés (doivent tenir en RAM pour performances optimales).

💡 **Principe** : Si working set > RAM → swap → dégradation performance.

🎯 **Dimensionnement** : RAM ≥ working set + cache WiredTiger.

---

### IOPS **[Avancé]**
🔤 **Input/Output Operations Per Second** - Nombre d'opérations de lecture/écriture disque par seconde.

💡 **Impact** : Métrique critique pour la performance, surtout si données > RAM.

🎯 **Production** : Préférer SSD (NVMe) pour IOPS élevés.

---

## Navigation rapide

- **[← Retour au sommaire du glossaire](README.md)**
- **[→ A.2 - Acronymes courants](02-acronymes-courants.md)**

---

**💡 Astuce** : Utilisez Ctrl+F (ou Cmd+F) pour rechercher rapidement un terme spécifique dans cette page.

**📚 Pour approfondir** : Chaque terme renvoie vers les chapitres détaillés de la formation principale.

⏭️ [Acronymes courants (CAP, ACID, TTL, WiredTiger, etc.)](/annexes/glossaire/02-acronymes-courants.md)
