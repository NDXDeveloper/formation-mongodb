🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22. Dépannage et Résolution de Problèmes MongoDB

## Vue d'ensemble

Le dépannage et la résolution de problèmes constituent une compétence essentielle pour tout administrateur MongoDB en environnement de production. Ce chapitre fournit des méthodologies systématiques, des guides de diagnostic pas à pas et des solutions éprouvées pour les problèmes les plus fréquemment rencontrés dans les déploiements MongoDB.

---

## Objectifs du chapitre

À l'issue de ce chapitre, vous serez capable de :

- **Diagnostiquer rapidement** les problèmes affectant MongoDB en production
- **Appliquer une méthodologie systématique** de résolution de problèmes
- **Interpréter les logs** et métriques pour identifier les causes racines
- **Résoudre efficacement** les incidents courants (connexion, performance, réplication, sharding)
- **Mettre en œuvre des procédures** de récupération après panne
- **Utiliser les outils** MongoDB pour l'analyse et le diagnostic
- **Escalader intelligemment** vers le support MongoDB quand nécessaire

---

## Méthodologie Générale de Dépannage

### Approche Structurée en 6 Étapes

```
1. IDENTIFIER
   └─ Collecter les symptômes et manifestations du problème

2. ISOLER
   └─ Reproduire le problème et circonscrire le périmètre

3. ANALYSER
   └─ Examiner logs, métriques et configurations

4. DIAGNOSTIQUER
   └─ Identifier la cause racine

5. RÉSOUDRE
   └─ Appliquer la solution et valider

6. DOCUMENTER
   └─ Consigner l'incident et les actions correctives
```

### Principe du "5 Whys" (5 Pourquoi)

Technique d'analyse pour remonter à la cause racine :

```
Symptôme : Les requêtes sont lentes
↓
Pourquoi ? → Les index ne sont pas utilisés
↓
Pourquoi ? → Le query planner choisit un mauvais plan
↓
Pourquoi ? → Les statistiques d'index sont obsolètes
↓
Pourquoi ? → La collection a été massivement modifiée
↓
Pourquoi ? → Pas de procédure de maintenance planifiée
→ SOLUTION : Mettre en place une maintenance automatique
```

---

## Outils Essentiels de Diagnostic

### 1. Commandes de Diagnostic Système

```javascript
// État général du serveur
db.serverStatus()

// Statistiques de base de données
db.stats()

// Statistiques de collection
db.collection.stats()

// Opérations en cours
db.currentOp()

// Historique des opérations lentes
db.system.profile.find().sort({ts: -1})

// État du Replica Set
rs.status()

// État du cluster shardé
sh.status()

// Configuration du serveur
db.adminCommand({getCmdLineOpts: 1})
```

### 2. Logs MongoDB

**Localisation par défaut des logs :**

```bash
# Linux
/var/log/mongodb/mongod.log

# macOS (Homebrew)
/usr/local/var/log/mongodb/mongod.log

# Windows
C:\Program Files\MongoDB\Server\<version>\log\mongod.log

# Docker
docker logs <container_id>
```

**Analyse en temps réel :**

```bash
# Suivre les logs en temps réel
tail -f /var/log/mongodb/mongod.log

# Filtrer les erreurs
grep -i "error" /var/log/mongodb/mongod.log

# Filtrer les opérations lentes (>100ms)
grep "Slow query" /var/log/mongodb/mongod.log

# Analyser les connexions
grep "connection" /var/log/mongodb/mongod.log
```

### 3. Outils en Ligne de Commande

```bash
# Statistiques en temps réel
mongostat --host <hostname>

# Top des opérations par collection
mongotop --host <hostname>

# Dump de diagnostic FTDC
db.adminCommand({getDiagnosticData: 1})

# Validation de collection
db.collection.validate({full: true})
```

### 4. MongoDB Compass

- **Visualisation des index** et leur utilisation
- **Analyse des requêtes** avec explain
- **Exploration des schémas** et validation
- **Monitoring en temps réel** des performances

### 5. MongoDB Cloud Manager / Ops Manager

- **Alertes automatiques** sur métriques critiques
- **Graphiques de performance** historiques
- **Analyse de logs** centralisée
- **Snapshots de diagnostic** automatiques

---

## Collecte d'Informations pour le Diagnostic

### Checklist de Collecte Initiale

Avant toute analyse approfondie, rassemblez systématiquement :

#### Information Système

```bash
# Version MongoDB
mongod --version

# Configuration système
uname -a
cat /etc/os-release

# Ressources disponibles
free -h
df -h
lscpu
iostat
vmstat 1 5
```

#### Information MongoDB

```javascript
// Version et configuration
db.version()
db.serverBuildInfo()
db.adminCommand({getParameter: "*"})

// État du déploiement
rs.conf()         // Pour Replica Set
sh.status()       // Pour Sharded Cluster

// Métriques actuelles
db.serverStatus().connections
db.serverStatus().opcounters
db.serverStatus().locks
db.serverStatus().wiredTiger
```

#### Logs Récents

```bash
# Dernières 1000 lignes
tail -n 1000 /var/log/mongodb/mongod.log

# Dernière heure
find /var/log/mongodb -name "mongod.log*" -mmin -60 -exec cat {} \;

# Erreurs et warnings uniquement
grep -E "(ERROR|WARN)" /var/log/mongodb/mongod.log | tail -n 100
```

---

## Matrice des Symptômes et Causes

### Table de Diagnostic Rapide

| Symptôme | Causes Probables | Priorité Investigation |
|----------|------------------|------------------------|
| **Connexions refusées** | • Limite maxConnections<br>• Firewall/réseau<br>• Authentification | 🔴 CRITIQUE |
| **Requêtes lentes** | • Manque d'index<br>• Lock contention<br>• Hardware sous-dimensionné | 🟡 HAUTE |
| **Utilisation mémoire élevée** | • Cache WiredTiger<br>• Requêtes inefficaces<br>• Memory leak | 🟡 HAUTE |
| **Utilisation disque élevée** | • Croissance des données<br>• Journalisation<br>• Oplog trop grand | 🟠 MOYENNE |
| **Réplication en retard** | • Réseau lent<br>• Opérations lourdes<br>• Secondary sous-dimensionné | 🔴 CRITIQUE |
| **Élections fréquentes** | • Réseau instable<br>• Heartbeat timeout<br>• Split-brain | 🔴 CRITIQUE |
| **Chunks non balancés** | • Balancer désactivé<br>• Mauvaise shard key<br>• Jumbo chunks | 🟠 MOYENNE |
| **Corruption de données** | • Arrêt brutal<br>• Défaillance disque<br>• Bug logiciel | 🔴 CRITIQUE |

---

## Niveaux de Sévérité et Temps de Réponse

### Classification des Incidents

#### 🔴 CRITIQUE (P1)

**Définition :** Service indisponible ou perte de données imminente

**Exemples :**
- Tous les nœuds d'un Replica Set sont down
- Impossibilité totale de connexion
- Corruption de données détectée
- Primary down sans élection de nouveau primary

**SLA de réponse :** 15 minutes
**SLA de résolution :** 2 heures

#### 🟡 HAUTE (P2)

**Définition :** Dégradation majeure de service

**Exemples :**
- Réplication lag > 1 heure
- Performance dégradée de 50%+
- Capacité disque > 90%
- Secondary members down

**SLA de réponse :** 1 heure
**SLA de résolution :** 8 heures

#### 🟠 MOYENNE (P3)

**Définition :** Problème affectant partiellement le service

**Exemples :**
- Requêtes spécifiques lentes
- Alertes de monitoring
- Balancer inefficace
- Utilisation mémoire élevée mais stable

**SLA de réponse :** 4 heures
**SLA de résolution :** 24 heures

#### 🟢 BASSE (P4)

**Définition :** Problème mineur ou demande d'amélioration

**Exemples :**
- Optimisation de requêtes
- Questions de configuration
- Demandes de documentation

**SLA de réponse :** 24 heures
**SLA de résolution :** 1 semaine

---

## Procédure d'Escalade

### Arbre de Décision pour l'Escalade

```
PROBLÈME DÉTECTÉ
     ↓
[Niveau L1 - Ops]
- Vérifications basiques
- Logs et métriques
- Solutions connues
     ↓
Résolu ? → OUI → FIN
     ↓ NON
[Niveau L2 - DBA Senior]
- Analyse approfondie
- Diagnostic système
- Modifications config
     ↓
Résolu ? → OUI → FIN
     ↓ NON
[Niveau L3 - Expert MongoDB]
- Analyse code/internals
- Patches temporaires
- Workarounds avancés
     ↓
Résolu ? → OUI → FIN
     ↓ NON
[Support MongoDB]
- Ticket avec diagnostic complet
- Logs et FTDC
- Collaboration ingénierie
```

### Informations à Fournir au Support MongoDB

Lorsque vous ouvrez un ticket support :

#### 1. Description du Problème

```
Titre : [Concis et descriptif]

Description :
- Symptômes observés
- Date/heure de début
- Fréquence (permanent/intermittent)
- Impact business
- Actions déjà entreprises
```

#### 2. Environnement

```
- Version MongoDB : X.X.X
- Système d'exploitation :
- Déploiement : Standalone/Replica Set/Sharded
- Nombre de nœuds :
- Hébergement : On-premise/Cloud/Atlas
- Drivers utilisés (versions) :
```

#### 3. Fichiers de Diagnostic

```bash
# Collecter le diagnostic complet
mongodump --archive --oplog
db.adminCommand({getDiagnosticData: 1})
db.serverStatus()
db.runCommand({buildInfo: 1})

# Logs (dernières 24h)
tar -czf logs.tar.gz /var/log/mongodb/

# Configuration
mongod --config /etc/mongod.conf --print-config
```

---

## Outils de Diagnostic Avancés

### 1. FTDC (Full Time Diagnostic Data Capture)

MongoDB collecte automatiquement des métriques système :

```javascript
// Localisation des fichiers FTDC
// Linux/macOS : <dbPath>/diagnostic.data/
// Windows : <dbPath>\diagnostic.data\

// Extraire les données FTDC
db.adminCommand({getDiagnosticData: 1})

// Activer/désactiver FTDC
db.adminCommand({
  setParameter: 1,
  diagnosticDataCollectionEnabled: true
})
```

**Utilisation :**
- Analyse post-mortem d'incidents
- Corrélation avec événements système
- Identification de patterns anormaux

### 2. Query Profiler

```javascript
// Activer le profiler (niveau 2 = toutes les requêtes)
db.setProfilingLevel(2)

// Niveau 1 = uniquement les requêtes lentes
db.setProfilingLevel(1, {slowms: 100})

// Analyser les requêtes les plus lentes
db.system.profile.find().sort({millis: -1}).limit(10)

// Analyser les requêtes par collection
db.system.profile.aggregate([
  {$group: {
    _id: "$ns",
    count: {$sum: 1},
    avgMillis: {$avg: "$millis"}
  }},
  {$sort: {avgMillis: -1}}
])

// Désactiver le profiler
db.setProfilingLevel(0)
```

### 3. Analyse de Locks

```javascript
// Vérifier les locks en cours
db.currentOp({
  $or: [
    {waitingForLock: true},
    {locks: {$exists: true}}
  ]
})

// Identifier les opérations bloquantes
db.currentOp().inprog.forEach(function(op) {
  if (op.waitingForLock) {
    printjson(op);
  }
})
```

### 4. Diagnostic Réseau

```bash
# Test de connectivité
telnet <hostname> 27017
nc -zv <hostname> 27017

# Test de latence
ping <hostname>
mtr <hostname>

# Vérifier les ports ouverts
netstat -tuln | grep 27017
ss -tuln | grep 27017

# Test de débit
iperf3 -c <hostname>
```

---

## Commandes d'Urgence

### Situations Critiques

#### Arrêt d'Urgence (à éviter si possible)

```bash
# Arrêt propre (préféré)
db.adminCommand({shutdown: 1})

# Ou via systemctl
sudo systemctl stop mongod

# En dernier recours (RISQUE DE CORRUPTION)
kill -9 <mongod_pid>  # À ÉVITER !
```

#### Libération de Connexions

```javascript
// Tuer une opération spécifique
db.killOp(<opid>)

// Tuer toutes les opérations longues
db.currentOp().inprog.forEach(function(op) {
  if (op.secs_running > 300) {
    db.killOp(op.opid);
  }
})

// Déconnecter un client spécifique
db.adminCommand({
  killAllSessionsByPattern: [
    {users: [{user: "username", db: "database"}]}
  ]
})
```

#### Compact en Urgence (libère de l'espace)

```javascript
// ATTENTION : Bloque les écritures
db.runCommand({compact: "collection"})

// Alternative sans blocage (migration vers nouveau shard)
// Pour Replica Set : rolling maintenance
```

#### Réparer une Base Corrompue

```bash
# ATTENTION : Dernière option, peut perdre des données
mongod --repair --dbpath /var/lib/mongodb

# Ou via commande
db.runCommand({repairDatabase: 1})
```

---

## Checklist de Maintenance Préventive

Prévenir plutôt que guérir - actions régulières recommandées :

### Quotidien

- [ ] Vérifier les alertes monitoring
- [ ] Vérifier l'utilisation disque
- [ ] Vérifier le replication lag
- [ ] Vérifier les erreurs dans les logs

### Hebdomadaire

- [ ] Analyser les requêtes lentes
- [ ] Vérifier l'utilisation des index
- [ ] Vérifier la croissance des données
- [ ] Tester les backups

### Mensuel

- [ ] Réviser les index (usage, redondance)
- [ ] Analyser les patterns d'accès
- [ ] Vérifier les versions de drivers
- [ ] Mettre à jour la documentation

### Trimestriel

- [ ] Planifier les mises à jour MongoDB
- [ ] Réviser l'architecture
- [ ] Tester les procédures de DR
- [ ] Audit de sécurité

---

## Structure de Documentation d'Incident

### Modèle de Rapport Post-Incident

```markdown
# Rapport d'Incident - [ID] - [Date]

## Résumé Exécutif
[Description en 2-3 phrases]

## Timeline
- **HH:MM** - Détection initiale
- **HH:MM** - Actions entreprises
- **HH:MM** - Service restauré

## Impact
- Durée : X heures
- Utilisateurs affectés : X
- Transactions perdues : X
- Coût estimé : X

## Cause Racine
[Analyse détaillée]

## Actions Correctives
1. Court terme :
   - [Action immédiate 1]
   - [Action immédiate 2]

2. Moyen terme :
   - [Amélioration 1]
   - [Amélioration 2]

3. Long terme :
   - [Changement structurel 1]
   - [Changement structurel 2]

## Leçons Apprises
- [Point 1]
- [Point 2]

## Actions de Suivi
| Action | Responsable | Date limite | Statut |
|--------|-------------|-------------|--------|
|        |             |             |        |
```

---

## Bonnes Pratiques Générales

### Principes de Dépannage Efficace

1. **Ne paniquez pas** - Une approche méthodique est toujours plus efficace
2. **Documentez tout** - Chaque action, chaque observation
3. **Une modification à la fois** - Pour identifier clairement l'impact
4. **Sauvegardez avant** - Toute modification de configuration ou données
5. **Testez en dev d'abord** - Quand c'est possible
6. **Communiquez** - Tenez informées toutes les parties prenantes
7. **Apprenez** - Chaque incident est une opportunité d'amélioration

### Ce qu'il NE faut PAS faire

❌ **Modifier plusieurs paramètres simultanément**
- Rend impossible l'identification de la solution réelle

❌ **Redémarrer sans analyse**
- Perd les données de diagnostic en mémoire

❌ **Exécuter des commandes de réparation sur primary en production**
- Risque de corruption et indisponibilité

❌ **Ignorer les warnings MongoDB**
- Les warnings précèdent souvent les erreurs critiques

❌ **Modifier directement les fichiers de données**
- Corruption de base garantie

❌ **Désactiver la sécurité temporairement**
- Peut créer des vulnérabilités permanentes

---

## Ressources et Références

### Documentation Officielle

- **MongoDB Manual** : https://docs.mongodb.com/manual/
- **MongoDB Support Portal** : https://support.mongodb.com/
- **MongoDB University** : https://university.mongodb.com/
- **MongoDB Community Forums** : https://community.mongodb.com/

### Outils Communautaires

- **mtools** : Outils d'analyse de logs et profiler
- **mongo-perf** : Framework de benchmarking
- **pt-mongodb-query-digest** : Analyse de requêtes (Percona)

### Contacts Support

```
Support MongoDB Enterprise :
- Portal : support.mongodb.com
- Email : support@mongodb.com
- Phone : [selon région]

Support Atlas :
- Depuis console Atlas
- Chat en ligne
- Tickets via portal
```

---

## Sections Détaillées du Chapitre

Ce chapitre se compose des sections suivantes, chacune fournissant des guides de diagnostic et résolution détaillés :

1. **[Problèmes de connexion](./01-problemes-connexion.md)**
   - Connexions refusées
   - Timeout de connexion
   - Épuisement du pool de connexions
   - Problèmes d'authentification

2. **[Problèmes de performance](./02-problemes-performance.md)**
   - Requêtes lentes
   - Utilisation CPU élevée
   - Saturation mémoire
   - Goulots d'étranglement I/O

3. **[Problèmes de réplication](./03-problemes-replication.md)**
   - Replication lag
   - Élections fréquentes
   - Synchronisation échouée
   - Oplog insuffisant

4. **[Problèmes de sharding](./04-problemes-sharding.md)**
   - Balancing bloqué
   - Jumbo chunks
   - Requêtes scatter-gather
   - Hotspots sur shards

5. **[Corruption de données](./05-corruption-donnees.md)**
   - Détection de corruption
   - Validation de collections
   - Procédures de réparation
   - Récupération depuis backup

6. **[Récupération après panne](./06-recuperation-apres-panne.md)**
   - Procédures de démarrage
   - Récupération d'un Replica Set
   - Récupération d'un cluster shardé
   - Plans de disaster recovery

7. **[Analyse des logs d'erreurs](./07-analyse-logs-erreurs.md)**
   - Interprétation des messages
   - Patterns d'erreurs communs
   - Corrélation d'événements
   - Outils d'analyse automatisée

8. **[Support MongoDB et ressources](./08-support-mongodb-ressources.md)**
   - Ouverture de tickets
   - SLA de support
   - Ressources communautaires
   - Formation continue

9. **[Communauté et forums](./09-communaute-forums.md)**
   - Forums MongoDB
   - Stack Overflow
   - Groupes d'utilisateurs
   - Conférences et événements

---

## Conclusion

Le dépannage efficace de MongoDB nécessite :

- **Une compréhension approfondie** de l'architecture et du fonctionnement interne
- **Une méthodologie rigoureuse** d'investigation et de résolution
- **Une maîtrise des outils** de diagnostic et de monitoring
- **Une documentation précise** des incidents et solutions
- **Une amélioration continue** des procédures et de l'infrastructure

La clé du succès réside dans la **prévention** par une maintenance régulière, un monitoring proactif et des tests de procédures de récupération, plutôt que dans la réaction à des crises.

Les sections suivantes détaillent les procédures spécifiques pour chaque catégorie de problèmes, avec des guides pas à pas adaptés aux situations réelles de production.

---

**Prérequis pour ce chapitre :**
- Maîtrise des concepts MongoDB fondamentaux
- Expérience en administration système Linux/Unix
- Connaissance des architectures Replica Set et Sharded Cluster
- Accès aux outils de monitoring et logs

**Prochaine section :** 22.1 Problèmes de connexion

⏭️ [Problèmes de connexion](/22-depannage-resolution-problemes/01-problemes-connexion.md)
