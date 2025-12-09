🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.1 Problèmes de Connexion

## Vue d'ensemble

Les problèmes de connexion sont parmi les incidents les plus critiques car ils empêchent complètement l'accès à la base de données. Cette section fournit des procédures de diagnostic et résolution systématiques pour tous les types de problèmes de connexion MongoDB.

---

## Table des Matières

1. [Connexions Refusées](#1-connexions-refus%C3%A9es)
2. [Timeouts de Connexion](#2-timeouts-de-connexion)
3. [Épuisement du Pool de Connexions](#3-%C3%A9puisement-du-pool-de-connexions)
4. [Problèmes d'Authentification](#4-probl%C3%A8mes-dauthentification)
5. [Problèmes TLS/SSL](#5-probl%C3%A8mes-tlsssl)
6. [Problèmes DNS et Réseau](#6-probl%C3%A8mes-dns-et-r%C3%A9seau)
7. [Problèmes de Configuration](#7-probl%C3%A8mes-de-configuration)
8. [Checklist Globale](#8-checklist-globale-de-diagnostic)

---

## 1. Connexions Refusées

### Symptômes

```
Error: connect ECONNREFUSED <host>:27017
MongoNetworkError: failed to connect to server
Connection refused by server
```

### Causes Possibles

- MongoDB n'est pas démarré
- MongoDB écoute sur une mauvaise interface
- Pare-feu bloque les connexions
- Limite de connexions atteinte
- Bind IP mal configuré

---

### Diagnostic Pas à Pas

#### Étape 1 : Vérifier que MongoDB est en cours d'exécution

```bash
# Vérifier le statut du service
sudo systemctl status mongod

# Vérifier le processus
ps aux | grep mongod

# Vérifier les ports en écoute
netstat -tuln | grep 27017
# ou
ss -tuln | grep 27017

# Sur macOS
lsof -i :27017
```

**Interprétation :**
- Si aucun processus n'écoute sur 27017 → MongoDB n'est pas démarré
- Si le processus existe mais pas d'écoute réseau → Problème de bind

#### Étape 2 : Vérifier la configuration de bind

```bash
# Vérifier la configuration actuelle
cat /etc/mongod.conf | grep bindIp

# Ou depuis MongoDB
db.adminCommand({getCmdLineOpts: 1}).parsed.net
```

**Configurations possibles :**

```yaml
# Écoute uniquement sur localhost (par défaut)
net:
  bindIp: 127.0.0.1
  port: 27017

# Écoute sur toutes les interfaces (ATTENTION : Sécurité)
net:
  bindIp: 0.0.0.0
  port: 27017

# Écoute sur interfaces spécifiques
net:
  bindIp: 127.0.0.1,192.168.1.10
  port: 27017
```

#### Étape 3 : Vérifier les pare-feu

```bash
# Linux (iptables)
sudo iptables -L -n -v | grep 27017

# Linux (firewalld)
sudo firewall-cmd --list-all
sudo firewall-cmd --list-ports

# Vérifier si le port est accessible
telnet <hostname> 27017
# ou
nc -zv <hostname> 27017

# Test depuis l'extérieur
nmap -p 27017 <hostname>
```

#### Étape 4 : Vérifier les logs MongoDB

```bash
# Voir les dernières erreurs
tail -n 100 /var/log/mongodb/mongod.log | grep -i "error\|fail"

# Rechercher des messages sur le bind
grep "bindIp" /var/log/mongodb/mongod.log

# Vérifier les messages de démarrage
grep "waiting for connections" /var/log/mongodb/mongod.log
```

#### Étape 5 : Tester la connexion locale

```bash
# Connexion locale (doit toujours fonctionner)
mongosh --host 127.0.0.1

# Connexion via hostname
mongosh --host $(hostname)

# Connexion via IP externe
mongosh --host <ip_externe>
```

---

### Résolution Pas à Pas

#### Solution 1 : Démarrer MongoDB

```bash
# Démarrer le service
sudo systemctl start mongod

# Vérifier le statut
sudo systemctl status mongod

# Activer le démarrage automatique
sudo systemctl enable mongod

# Vérifier les logs après démarrage
tail -f /var/log/mongodb/mongod.log
```

**Si le démarrage échoue :**

```bash
# Vérifier les permissions
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb

# Vérifier l'espace disque
df -h /var/lib/mongodb

# Tenter un démarrage manuel pour voir les erreurs
sudo -u mongodb mongod --config /etc/mongod.conf
```

#### Solution 2 : Corriger la configuration de bind

**Scénario : Autoriser les connexions distantes**

```bash
# 1. Éditer la configuration
sudo nano /etc/mongod.conf

# 2. Modifier la section net
net:
  bindIp: 0.0.0.0  # ou liste d'IPs spécifiques
  port: 27017

# 3. Redémarrer MongoDB
sudo systemctl restart mongod

# 4. Vérifier la nouvelle configuration
netstat -tuln | grep 27017
```

**Sortie attendue :**
```
tcp    0    0 0.0.0.0:27017    0.0.0.0:*    LISTEN
```

**⚠️ Important :** N'exposez jamais MongoDB sans authentification sur 0.0.0.0 en production.

#### Solution 3 : Configurer le pare-feu

**Linux avec firewalld :**

```bash
# Ouvrir le port 27017
sudo firewall-cmd --permanent --add-port=27017/tcp

# Pour Replica Set (27017-27019)
sudo firewall-cmd --permanent --add-port=27017-27019/tcp

# Recharger la configuration
sudo firewall-cmd --reload

# Vérifier
sudo firewall-cmd --list-ports
```

**Linux avec iptables :**

```bash
# Autoriser le port 27017
sudo iptables -A INPUT -p tcp --dport 27017 -j ACCEPT

# Autoriser depuis une IP spécifique
sudo iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 27017 -j ACCEPT

# Sauvegarder les règles
sudo iptables-save > /etc/iptables/rules.v4
```

**AWS Security Groups :**

```bash
# Via AWS CLI
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp \
  --port 27017 \
  --cidr 10.0.0.0/8
```

**Vérifier l'ouverture du port :**

```bash
# Depuis un autre serveur
telnet <mongodb_host> 27017

# Si telnet n'est pas disponible
timeout 5 bash -c "</dev/tcp/<mongodb_host>/27017" && echo "Port open" || echo "Port closed"
```

#### Solution 4 : Augmenter la limite de connexions

```bash
# Vérifier la limite actuelle
db.serverStatus().connections

# Sortie exemple :
# {
#   "current": 945,
#   "available": 55,
#   "totalCreated": 12450
# }
```

**Si available est faible, augmenter maxIncomingConnections :**

```yaml
# Dans /etc/mongod.conf
net:
  maxIncomingConnections: 65536

# Ou via setParameter
db.adminCommand({
  setParameter: 1,
  maxIncomingConnections: 65536
})
```

**Augmenter les limites système (Linux) :**

```bash
# Vérifier les limites actuelles
ulimit -n

# Augmenter temporairement
ulimit -n 64000

# Augmenter de façon permanente
sudo nano /etc/security/limits.conf

# Ajouter :
mongodb soft nofile 64000
mongodb hard nofile 64000
```

---

## 2. Timeouts de Connexion

### Symptômes

```
MongoNetworkError: connection timeout
Error: connect ETIMEDOUT
Server selection timeout after 30000 ms
```

### Causes Possibles

- Latence réseau élevée
- Pare-feu avec inspection de paquets lente
- MongoDB surchargé
- Configuration timeout trop basse
- DNS lent

---

### Diagnostic Pas à Pas

#### Étape 1 : Mesurer la latence réseau

```bash
# Ping simple
ping -c 10 <mongodb_host>

# Trace route
traceroute <mongodb_host>
# ou
mtr <mongodb_host>

# Test de latence TCP sur port 27017
hping3 -S -p 27017 -c 10 <mongodb_host>
```

**Seuils de latence acceptables :**
- < 1ms : Excellent (même datacenter)
- 1-10ms : Bon (même région)
- 10-50ms : Acceptable (inter-régions proches)
- 50-100ms : Moyen (inter-continents)
- > 100ms : Problématique pour certaines opérations

#### Étape 2 : Tester la connexion TCP

```bash
# Test de connexion avec timeout personnalisé
timeout 5 bash -c "cat < /dev/null > /dev/tcp/<host>/27017" && echo "Success" || echo "Failed"

# Test avec telnet chronométré
time telnet <mongodb_host> 27017

# Test avec nc (netcat)
nc -zv -w5 <mongodb_host> 27017
```

#### Étape 3 : Vérifier la charge MongoDB

```bash
# Connexion si possible
mongosh --host <mongodb_host>

# Vérifier les opérations en cours
db.currentOp({
  $or: [
    {secs_running: {$gt: 30}},
    {waitingForLock: true}
  ]
})

# Vérifier les métriques serveur
db.serverStatus().connections
db.serverStatus().opcounters
```

#### Étape 4 : Tester la résolution DNS

```bash
# Temps de résolution DNS
time nslookup <mongodb_host>
time dig <mongodb_host>

# Vérifier /etc/hosts
cat /etc/hosts | grep <mongodb_host>

# Tester avec IP directe (contourne DNS)
mongosh --host <ip_address>
```

#### Étape 5 : Analyser les logs réseau

```bash
# Capturer le trafic sur port 27017
sudo tcpdump -i any port 27017 -w /tmp/mongodb.pcap

# Analyser avec wireshark ou tshark
tshark -r /tmp/mongodb.pcap -Y "tcp.port==27017"

# Vérifier les TCP retransmissions
tshark -r /tmp/mongodb.pcap -Y "tcp.analysis.retransmission"
```

---

### Résolution Pas à Pas

#### Solution 1 : Augmenter les timeouts applicatifs

**Node.js (MongoDB Driver) :**

```javascript
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 60000,    // 60 secondes (défaut: 30s)
  connectTimeoutMS: 30000,             // 30 secondes (défaut: 10s)
  socketTimeoutMS: 60000,              // 60 secondes (défaut: pas de timeout)
  heartbeatFrequencyMS: 10000,         // 10 secondes (défaut: 10s)
});
```

**Python (PyMongo) :**

```python
client = MongoClient(
    uri,
    serverSelectionTimeoutMS=60000,
    connectTimeoutMS=30000,
    socketTimeoutMS=60000,
)
```

**Java :**

```java
MongoClientSettings settings = MongoClientSettings.builder()
    .applyToClusterSettings(builder ->
        builder.serverSelectionTimeout(60, TimeUnit.SECONDS))
    .applyToSocketSettings(builder ->
        builder.connectTimeout(30, TimeUnit.SECONDS)
               .readTimeout(60, TimeUnit.SECONDS))
    .build();
```

**Connection String URI :**

```
mongodb://host:27017/db?serverSelectionTimeoutMS=60000&connectTimeoutMS=30000&socketTimeoutMS=60000
```

#### Solution 2 : Optimiser la configuration réseau

**TCP Keepalive (MongoDB) :**

```yaml
# /etc/mongod.conf
net:
  tcpKeepAlive: true
```

**TCP Keepalive (Système Linux) :**

```bash
# Vérifier les valeurs actuelles
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes

# Optimiser pour MongoDB (réduit les timeouts)
sudo sysctl -w net.ipv4.tcp_keepalive_time=120
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=30
sudo sysctl -w net.ipv4.tcp_keepalive_probes=3

# Rendre permanent
sudo nano /etc/sysctl.conf
# Ajouter :
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

# Appliquer
sudo sysctl -p
```

#### Solution 3 : Optimiser la résolution DNS

**Ajouter des entrées /etc/hosts :**

```bash
sudo nano /etc/hosts

# Ajouter les hôtes MongoDB
192.168.1.10  mongodb-primary
192.168.1.11  mongodb-secondary1
192.168.1.12  mongodb-secondary2
```

**Configurer un cache DNS local (dnsmasq) :**

```bash
# Installer dnsmasq
sudo apt-get install dnsmasq

# Configuration
sudo nano /etc/dnsmasq.conf
# Ajouter :
cache-size=1000
no-negcache

# Redémarrer
sudo systemctl restart dnsmasq
```

**Utiliser IP directement (contournement temporaire) :**

```javascript
// Au lieu de
mongodb://mongodb-primary:27017,mongodb-secondary:27017

// Utiliser
mongodb://192.168.1.10:27017,192.168.1.11:27017
```

#### Solution 4 : Optimiser MongoDB surchargé

```javascript
// Identifier les opérations lentes
db.currentOp({
  active: true,
  secs_running: {$gt: 10}
})

// Tuer les opérations problématiques
db.killOp(<opid>)

// Vérifier les index manquants
db.collection.aggregate([
  {$indexStats: {}},
  {$sort: {accesses: -1}}
])

// Redémarrer avec paramètres de performance
// (à faire lors d'une fenêtre de maintenance)
```

---

## 3. Épuisement du Pool de Connexions

### Symptômes

```
MongoError: connection pool destroyed
MongoError: no connection available
Error: Pool is draining
Too many connections
```

### Causes Possibles

- Fuites de connexions dans le code
- Pool size trop petit
- Connexions non fermées explicitement
- Opérations longues monopolisant les connexions
- Plusieurs instances d'application avec pools mal dimensionnés

---

### Diagnostic Pas à Pas

#### Étape 1 : Vérifier l'utilisation des connexions

```javascript
// Côté MongoDB
db.serverStatus().connections
// Résultat :
// {
//   current: 950,      // Connexions actuelles
//   available: 50,     // Disponibles
//   totalCreated: 15000 // Total créées depuis démarrage
// }

// Identifier les clients avec le plus de connexions
db.aggregate([
  {$currentOp: {allUsers: true, idleConnections: true}},
  {$group: {
    _id: "$client",
    count: {$sum: 1}
  }},
  {$sort: {count: -1}}
])
```

#### Étape 2 : Analyser les pools applicatifs

**Node.js :**

```javascript
// Ajouter du monitoring
client.on('connectionPoolCreated', (event) => {
  console.log('Pool created:', event);
});

client.on('connectionCreated', (event) => {
  console.log('Connection created:', event.connectionId);
});

client.on('connectionClosed', (event) => {
  console.log('Connection closed:', event.connectionId);
});

client.on('connectionCheckOutStarted', (event) => {
  console.log('Checkout started');
});

client.on('connectionCheckOutFailed', (event) => {
  console.error('Checkout failed:', event.reason);
});

// Monitorer l'état du pool
setInterval(() => {
  const stats = client.topology.s.pool.stats();
  console.log('Pool stats:', stats);
}, 5000);
```

**Python :**

```python
from pymongo import monitoring

class CommandLogger(monitoring.CommandListener):
    def started(self, event):
        print(f"Command started: {event.command_name}")

    def succeeded(self, event):
        print(f"Command succeeded: {event.command_name}")

    def failed(self, event):
        print(f"Command failed: {event.command_name}")

monitoring.register(CommandLogger())
```

#### Étape 3 : Identifier les fuites de connexions

```bash
# Monitorer les connexions actives dans le temps
watch -n 1 'mongosh --quiet --eval "db.serverStatus().connections"'

# Compter les connexions par IP source
netstat -tn | grep :27017 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Identifier les connexions idle longues
db.aggregate([
  {$currentOp: {allUsers: true, idleConnections: true}},
  {$match: {
    "active": false,
    "secs_running": {$gt: 300}
  }},
  {$project: {
    client: 1,
    connectionId: 1,
    secs_running: 1,
    desc: 1
  }},
  {$sort: {secs_running: -1}}
])
```

#### Étape 4 : Vérifier la configuration des pools

```javascript
// Vérifier la configuration actuelle
console.log('Max Pool Size:', client.options.maxPoolSize);
console.log('Min Pool Size:', client.options.minPoolSize);
console.log('Wait Queue Timeout:', client.options.waitQueueTimeoutMS);
```

---

### Résolution Pas à Pas

#### Solution 1 : Dimensionner correctement le pool

**Formule de dimensionnement :**

```
Pool Size = (Nombre de requêtes concurrentes moyennes) × (Temps de réponse moyen en secondes) + Marge (20%)
```

**Exemple :**
- 100 requêtes/sec
- Temps de réponse : 50ms (0.05s)
- Pool minimal : 100 × 0.05 + 20% = 6 connexions

**Configuration Node.js :**

```javascript
const client = new MongoClient(uri, {
  maxPoolSize: 100,              // Maximum (défaut: 100)
  minPoolSize: 10,               // Minimum (défaut: 0)
  maxIdleTimeMS: 30000,          // Fermer après 30s d'inactivité
  waitQueueTimeoutMS: 10000,     // Timeout attente connexion disponible
});
```

**Configuration Python :**

```python
client = MongoClient(
    uri,
    maxPoolSize=100,
    minPoolSize=10,
    maxIdleTimeMS=30000,
    waitQueueTimeoutMS=10000,
)
```

**Configuration Java :**

```java
MongoClientSettings settings = MongoClientSettings.builder()
    .applyToConnectionPoolSettings(builder ->
        builder.maxSize(100)
               .minSize(10)
               .maxWaitTime(10, TimeUnit.SECONDS)
               .maxConnectionIdleTime(30, TimeUnit.SECONDS))
    .build();
```

#### Solution 2 : Corriger les fuites de connexions

**❌ Code avec fuite :**

```javascript
// MAUVAIS : Crée une nouvelle connexion à chaque requête
app.get('/users', async (req, res) => {
  const client = new MongoClient(uri);
  await client.connect();
  const users = await client.db().collection('users').find().toArray();
  res.json(users);
  // Connexion jamais fermée !
});
```

**✅ Code correct :**

```javascript
// BON : Réutilise la même connexion
const client = new MongoClient(uri);
await client.connect(); // Une seule fois au démarrage

app.get('/users', async (req, res) => {
  const users = await client.db().collection('users').find().toArray();
  res.json(users);
});

// Fermer proprement à l'arrêt
process.on('SIGINT', async () => {
  await client.close();
  process.exit(0);
});
```

**Pattern Singleton (Node.js) :**

```javascript
// db.js
let client = null;

export async function getDatabase() {
  if (!client) {
    client = new MongoClient(uri, options);
    await client.connect();
  }
  return client.db();
}

export async function closeDatabase() {
  if (client) {
    await client.close();
    client = null;
  }
}
```

#### Solution 3 : Implémenter la gestion du cycle de vie

**Express.js avec middleware :**

```javascript
const client = new MongoClient(uri);

async function startServer() {
  try {
    await client.connect();
    console.log('Connected to MongoDB');

    app.listen(3000, () => {
      console.log('Server started');
    });
  } catch (err) {
    console.error('Failed to connect to MongoDB', err);
    process.exit(1);
  }
}

// Graceful shutdown
async function shutdown() {
  console.log('Shutting down gracefully...');

  // Arrêter d'accepter de nouvelles connexions
  server.close(() => {
    console.log('HTTP server closed');
  });

  // Fermer la connexion MongoDB
  await client.close();
  console.log('MongoDB connection closed');

  process.exit(0);
}

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

startServer();
```

#### Solution 4 : Limiter les connexions côté MongoDB

```javascript
// Augmenter la limite côté serveur
db.adminCommand({
  setParameter: 1,
  maxIncomingConnections: 65536
})

// Fermer les connexions idle de force
db.aggregate([
  {$currentOp: {allUsers: true, idleConnections: true}},
  {$match: {
    active: false,
    secs_running: {$gt: 600}  // Plus de 10 minutes idle
  }}
]).forEach(op => {
  try {
    db.killOp(op.opid);
  } catch (e) {
    print('Could not kill operation: ' + op.opid);
  }
});
```

#### Solution 5 : Monitorer et alerter

**Créer des alertes sur l'utilisation des connexions :**

```javascript
// Script de monitoring
function checkConnectionHealth() {
  const stats = db.serverStatus().connections;
  const usagePercent = (stats.current / (stats.current + stats.available)) * 100;

  if (usagePercent > 90) {
    console.error(`CRITICAL: Connection usage at ${usagePercent.toFixed(2)}%`);
    // Envoyer alerte (email, PagerDuty, etc.)
  } else if (usagePercent > 75) {
    console.warn(`WARNING: Connection usage at ${usagePercent.toFixed(2)}%`);
  }

  return {
    current: stats.current,
    available: stats.available,
    usagePercent: usagePercent.toFixed(2)
  };
}

// Exécuter toutes les 30 secondes
setInterval(checkConnectionHealth, 30000);
```

---

## 4. Problèmes d'Authentification

### Symptômes

```
MongoServerError: Authentication failed
Error: Authentication failed
MongoError: bad auth
MongoError: not authorized
```

### Causes Possibles

- Identifiants incorrects
- Base de données d'authentification incorrecte
- Mécanisme d'authentification non supporté
- Utilisateur n'existe pas ou désactivé
- Problèmes de rôles et privilèges

---

### Diagnostic Pas à Pas

#### Étape 1 : Vérifier la configuration d'authentification

```bash
# Vérifier si l'authentification est activée
grep "security:" /etc/mongod.conf

# Sortie attendue :
# security:
#   authorization: enabled
```

```javascript
// Depuis MongoDB shell
db.adminCommand({getParameter: 1, authenticationMechanisms: 1})

// Vérifier si l'authentification est requise
db.adminCommand({connectionStatus: 1}).authInfo
```

#### Étape 2 : Vérifier l'existence de l'utilisateur

```javascript
// Connexion en tant qu'admin
mongosh --host localhost --authenticationDatabase admin -u admin

// Lister tous les utilisateurs
use admin
db.system.users.find({}, {user: 1, db: 1, roles: 1})

// Vérifier un utilisateur spécifique
use mydb
db.getUser("username")
```

#### Étape 3 : Vérifier la string de connexion

**Format complet :**

```
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/[defaultauthdb][?options]]
```

**Exemples :**

```bash
# Authentification sur base admin (par défaut)
mongodb://user:password@localhost:27017/mydb?authSource=admin

# Authentification sur base spécifique
mongodb://user:password@localhost:27017/mydb?authSource=mydb

# Avec caractères spéciaux (encode URI)
mongodb://user:p%40ssw%40rd@localhost:27017/mydb
```

#### Étape 4 : Tester l'authentification manuellement

```bash
# Test basique
mongosh --host localhost -u username -p password --authenticationDatabase admin

# Test avec différentes bases d'authentification
mongosh "mongodb://user:pass@localhost/mydb?authSource=admin"
mongosh "mongodb://user:pass@localhost/mydb?authSource=mydb"

# Test avec mécanisme spécifique
mongosh "mongodb://user:pass@localhost/mydb?authMechanism=SCRAM-SHA-256"
```

#### Étape 5 : Vérifier les logs

```bash
# Rechercher les erreurs d'authentification
grep -i "auth" /var/log/mongodb/mongod.log | tail -n 50

# Messages spécifiques à rechercher :
# - "authentication failed"
# - "SCRAM-SHA-256 authentication failed"
# - "no such user"
# - "not authorized"
```

---

### Résolution Pas à Pas

#### Solution 1 : Créer ou recréer l'utilisateur

```javascript
// Connexion en mode admin
mongosh

// Créer un utilisateur admin (si nécessaire)
use admin
db.createUser({
  user: "admin",
  pwd: "securePassword123",
  roles: [
    {role: "userAdminAnyDatabase", db: "admin"},
    {role: "dbAdminAnyDatabase", db: "admin"},
    {role: "readWriteAnyDatabase", db: "admin"}
  ]
})

// Créer un utilisateur pour une base spécifique
use mydb
db.createUser({
  user: "appuser",
  pwd: "appPassword456",
  roles: [
    {role: "readWrite", db: "mydb"}
  ]
})

// Vérifier la création
db.getUser("appuser")
```

#### Solution 2 : Réinitialiser le mot de passe

```javascript
// Connexion en tant qu'admin
use admin
mongosh -u admin -p

// Changer le mot de passe
use mydb
db.changeUserPassword("username", "newSecurePassword")

// Ou avec updateUser
db.updateUser("username", {
  pwd: "newSecurePassword"
})

// Vérifier
db.auth("username", "newSecurePassword")
```

#### Solution 3 : Corriger les rôles et privilèges

```javascript
// Voir les rôles actuels
db.getUser("username").roles

// Ajouter des rôles
db.grantRolesToUser("username", [
  {role: "readWrite", db: "mydb"},
  {role: "dbAdmin", db: "mydb"}
])

// Supprimer des rôles
db.revokeRolesFromUser("username", [
  {role: "read", db: "mydb"}
])

// Remplacer tous les rôles
db.updateUser("username", {
  roles: [
    {role: "readWrite", db: "mydb"},
    {role: "dbAdmin", db: "mydb"}
  ]
})
```

#### Solution 4 : Désactiver temporairement l'authentification (urgence)

**⚠️ ATTENTION : Seulement en environnement de développement ou urgence absolue**

```bash
# 1. Arrêter MongoDB
sudo systemctl stop mongod

# 2. Démarrer sans authentification
mongod --dbpath /var/lib/mongodb --noauth --port 27017

# 3. Dans un autre terminal, corriger les utilisateurs
mongosh --port 27017

use admin
// Créer/modifier utilisateurs

# 4. Arrêter et redémarrer normalement
sudo systemctl start mongod
```

#### Solution 5 : Configurer l'authentification pour la première fois

```bash
# 1. MongoDB installé sans authentification
mongosh

# 2. Créer le premier utilisateur admin
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),  // Demande le mot de passe de façon sécurisée
  roles: [
    {role: "userAdminAnyDatabase", db: "admin"},
    {role: "readWriteAnyDatabase", db: "admin"}
  ]
})

# 3. Déconnecter et activer l'authentification
exit

# 4. Éditer la configuration
sudo nano /etc/mongod.conf

# Ajouter :
security:
  authorization: enabled

# 5. Redémarrer MongoDB
sudo systemctl restart mongod

# 6. Tester la connexion
mongosh -u admin -p --authenticationDatabase admin
```

#### Solution 6 : Gérer les caractères spéciaux dans les mots de passe

```javascript
// Si le mot de passe contient des caractères spéciaux
// Exemple : p@ssw0rd!123

// Option 1 : Encoder en URI
const password = encodeURIComponent("p@ssw0rd!123");
const uri = `mongodb://user:${password}@localhost/mydb`;

// Option 2 : Utiliser une variable d'environnement
// .env
MONGO_PASSWORD=p@ssw0rd!123

// Code
const uri = `mongodb://user:${process.env.MONGO_PASSWORD}@localhost/mydb`;

// Option 3 : Configuration séparée (MongoDB Driver)
const client = new MongoClient('mongodb://localhost', {
  auth: {
    username: 'user',
    password: 'p@ssw0rd!123'
  }
});
```

---

## 5. Problèmes TLS/SSL

### Symptômes

```
MongoError: SSL routines
Error: certificate verify failed
DEPTH_ZERO_SELF_SIGNED_CERT
UNABLE_TO_VERIFY_LEAF_SIGNATURE
```

### Causes Possibles

- Certificat auto-signé sans configuration appropriée
- Certificat expiré
- Nom d'hôte ne correspond pas au certificat
- CA non reconnu
- Configuration TLS/SSL mal configurée

---

### Diagnostic Pas à Pas

#### Étape 1 : Vérifier la configuration TLS/SSL

```bash
# Vérifier la configuration MongoDB
cat /etc/mongod.conf | grep -A 10 "net:"

# Sortie attendue :
# net:
#   tls:
#     mode: requireTLS
#     certificateKeyFile: /path/to/cert.pem
#     CAFile: /path/to/ca.pem
```

```javascript
// Depuis MongoDB
db.adminCommand({getCmdLineOpts: 1}).parsed.net.tls
```

#### Étape 2 : Vérifier le certificat

```bash
# Voir les détails du certificat
openssl x509 -in /path/to/cert.pem -text -noout

# Vérifier la date d'expiration
openssl x509 -in /path/to/cert.pem -noout -enddate

# Vérifier le Common Name (CN) et Subject Alternative Names (SAN)
openssl x509 -in /path/to/cert.pem -noout -subject -ext subjectAltName

# Vérifier la chaîne de certificats
openssl verify -CAfile /path/to/ca.pem /path/to/cert.pem
```

#### Étape 3 : Tester la connexion TLS

```bash
# Test avec openssl
openssl s_client -connect <hostname>:27017 -CAfile /path/to/ca.pem

# Test avec mongosh
mongosh "mongodb://<hostname>:27017?tls=true&tlsCAFile=/path/to/ca.pem"

# Test sans vérification (DEBUG UNIQUEMENT)
mongosh "mongodb://<hostname>:27017?tls=true&tlsAllowInvalidCertificates=true"
```

#### Étape 4 : Vérifier les logs TLS

```bash
# Logs MongoDB
grep -i "ssl\|tls" /var/log/mongodb/mongod.log

# Messages à rechercher :
# - "SSL handshake failed"
# - "certificate verify failed"
# - "SSL peer certificate validation failed"
```

---

### Résolution Pas à Pas

#### Solution 1 : Configurer TLS correctement

**Configuration MongoDB (requireTLS) :**

```yaml
# /etc/mongod.conf
net:
  port: 27017
  bindIp: 0.0.0.0
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/mongodb.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowConnectionsWithoutCertificates: false  # Client cert requis
    allowInvalidCertificates: false              # Validation stricte
```

**Configuration MongoDB (allowTLS - optionnel) :**

```yaml
net:
  tls:
    mode: allowTLS  # Accepte TLS et non-TLS
    certificateKeyFile: /etc/ssl/mongodb/mongodb.pem
```

#### Solution 2 : Générer un nouveau certificat

**Certificat auto-signé (développement uniquement) :**

```bash
# Générer une clé privée
openssl genrsa -out mongodb.key 4096

# Générer un certificat auto-signé
openssl req -new -x509 -days 365 -key mongodb.key -out mongodb.crt \
  -subj "/C=FR/ST=IDF/L=Paris/O=MyCompany/CN=mongodb.local"

# Combiner clé et certificat
cat mongodb.key mongodb.crt > mongodb.pem

# Définir les permissions
chmod 400 mongodb.pem
chown mongodb:mongodb mongodb.pem
```

**Certificat Let's Encrypt (production) :**

```bash
# Installer certbot
sudo apt-get install certbot

# Obtenir un certificat
sudo certbot certonly --standalone -d mongodb.example.com

# Certificats générés dans :
# /etc/letsencrypt/live/mongodb.example.com/

# Créer le PEM pour MongoDB
sudo cat /etc/letsencrypt/live/mongodb.example.com/privkey.pem \
         /etc/letsencrypt/live/mongodb.example.com/fullchain.pem \
         > /etc/ssl/mongodb/mongodb.pem

# Permissions
sudo chmod 400 /etc/ssl/mongodb/mongodb.pem
sudo chown mongodb:mongodb /etc/ssl/mongodb/mongodb.pem

# Auto-renouvellement
sudo certbot renew --dry-run
```

#### Solution 3 : Configurer le client pour accepter le certificat

**Node.js :**

```javascript
// Avec validation complète
const client = new MongoClient(uri, {
  tls: true,
  tlsCAFile: '/path/to/ca.pem',
  tlsCertificateKeyFile: '/path/to/client.pem',
  tlsAllowInvalidHostnames: false,
  tlsAllowInvalidCertificates: false,
});

// En développement (certificat auto-signé)
const client = new MongoClient(uri, {
  tls: true,
  tlsAllowInvalidCertificates: true,  // Accepte les certificats auto-signés
  tlsAllowInvalidHostnames: true,      // Ignore la vérification du hostname
});
```

**Python :**

```python
# Avec validation
client = MongoClient(
    'mongodb://hostname:27017',
    tls=True,
    tlsCAFile='/path/to/ca.pem',
    tlsCertificateKeyFile='/path/to/client.pem',
)

# Sans validation (dev uniquement)
client = MongoClient(
    'mongodb://hostname:27017',
    tls=True,
    tlsAllowInvalidCertificates=True,
)
```

**Connection String :**

```
mongodb://hostname:27017/db?tls=true&tlsCAFile=/path/to/ca.pem&tlsCertificateKeyFile=/path/to/client.pem
```

#### Solution 4 : Ajouter le CA au trust store système

**Linux (Debian/Ubuntu) :**

```bash
# Copier le CA
sudo cp ca.crt /usr/local/share/ca-certificates/mongodb-ca.crt

# Mettre à jour le trust store
sudo update-ca-certificates

# Vérifier
openssl verify -CApath /etc/ssl/certs /path/to/mongodb.crt
```

**Linux (RedHat/CentOS) :**

```bash
# Copier le CA
sudo cp ca.crt /etc/pki/ca-trust/source/anchors/mongodb-ca.crt

# Mettre à jour
sudo update-ca-trust extract

# Vérifier
openssl verify -CApath /etc/pki/tls/certs /path/to/mongodb.crt
```

#### Solution 5 : Renouveler un certificat expiré

```bash
# 1. Vérifier l'expiration
openssl x509 -in /etc/ssl/mongodb/mongodb.pem -noout -dates

# 2. Générer un nouveau certificat (voir Solution 2)

# 3. Remplacer sans interruption (Replica Set)
# Sur chaque membre, un par un :

# a. Remplacer les fichiers
sudo cp new-mongodb.pem /etc/ssl/mongodb/mongodb.pem
sudo chmod 400 /etc/ssl/mongodb/mongodb.pem
sudo chown mongodb:mongodb /etc/ssl/mongodb/mongodb.pem

# b. Recharger la configuration (rolling restart)
db.adminCommand({rotateCertificates: 1})

# c. Vérifier
db.adminCommand({getCmdLineOpts: 1})

# 4. Mettre à jour les clients avec le nouveau CA si nécessaire
```

---

## 6. Problèmes DNS et Réseau

### Symptômes

```
MongoServerError: getaddrinfo ENOTFOUND
Error: No server available
MongoNetworkError: failed to resolve DNS
```

### Causes Possibles

- Serveur DNS indisponible ou lent
- Configuration DNS incorrecte
- Problème de routage réseau
- Firewall bloquant les requêtes DNS
- Replica Set hostname non résolvable

---

### Diagnostic Pas à Pas

#### Étape 1 : Tester la résolution DNS

```bash
# Test basique
nslookup mongodb.example.com
host mongodb.example.com
dig mongodb.example.com

# Mesurer le temps de résolution
time nslookup mongodb.example.com

# Tester avec DNS spécifique
nslookup mongodb.example.com 8.8.8.8
dig @8.8.8.8 mongodb.example.com

# Vérifier la configuration DNS système
cat /etc/resolv.conf
```

#### Étape 2 : Tester la connectivité réseau

```bash
# Ping
ping -c 5 mongodb.example.com

# Traceroute
traceroute mongodb.example.com
mtr mongodb.example.com

# Test de port TCP
telnet mongodb.example.com 27017
nc -zv mongodb.example.com 27017

# Test avec IP (contourne DNS)
telnet <ip_address> 27017
```

#### Étape 3 : Vérifier la configuration Replica Set

```javascript
// Voir la configuration du Replica Set
rs.conf()

// Vérifier que tous les hostnames sont résolvables
rs.conf().members.forEach(member => {
  print("Testing: " + member.host);
  // Tester depuis le shell système
})
```

```bash
# Depuis le shell système
for host in mongodb1.example.com mongodb2.example.com mongodb3.example.com; do
  echo "Testing $host"
  nslookup $host
  ping -c 1 $host
done
```

#### Étape 4 : Diagnostiquer les problèmes de Replica Set SRV

```bash
# Pour les connection strings SRV
# Format : mongodb+srv://cluster.example.com

# Vérifier les enregistrements SRV
dig +short SRV _mongodb._tcp.cluster.example.com

# Vérifier les enregistrements TXT (options)
dig +short TXT cluster.example.com
```

---

### Résolution Pas à Pas

#### Solution 1 : Utiliser /etc/hosts pour contourner DNS

```bash
# Éditer /etc/hosts
sudo nano /etc/hosts

# Ajouter les entrées
192.168.1.10  mongodb-primary mongodb-primary.local
192.168.1.11  mongodb-secondary1 mongodb-secondary1.local
192.168.1.12  mongodb-secondary2 mongodb-secondary2.local

# Sur chaque machine du Replica Set
# Ajouter toutes les autres machines

# Tester
ping mongodb-primary
```

#### Solution 2 : Configurer un DNS interne

**Avec dnsmasq :**

```bash
# Installation
sudo apt-get install dnsmasq

# Configuration
sudo nano /etc/dnsmasq.conf

# Ajouter :
address=/mongodb-primary.local/192.168.1.10
address=/mongodb-secondary1.local/192.168.1.11
address=/mongodb-secondary2.local/192.168.1.12

# Redémarrer
sudo systemctl restart dnsmasq

# Configurer comme DNS local
sudo nano /etc/resolv.conf
# Ajouter en première ligne :
nameserver 127.0.0.1
```

#### Solution 3 : Reconfigurer le Replica Set avec IP

**⚠️ Attention : Nécessite un reconfiguration complète**

```javascript
// 1. Sauvegarder la configuration actuelle
var config = rs.conf()
printjson(config)

// 2. Modifier les hostnames en IP
config.members[0].host = "192.168.1.10:27017"
config.members[1].host = "192.168.1.11:27017"
config.members[2].host = "192.168.1.12:27017"

// 3. Incrémenter la version
config.version++

// 4. Forcer la reconfiguration
rs.reconfig(config, {force: true})

// 5. Vérifier
rs.status()
```

#### Solution 4 : Utiliser directSeedList pour contourner SRV

```javascript
// Au lieu de mongodb+srv://
const client = new MongoClient('mongodb+srv://cluster.example.com/db');

// Utiliser une liste directe d'hôtes
const client = new MongoClient(
  'mongodb://host1:27017,host2:27017,host3:27017/db?replicaSet=rs0'
);

// Ou avec IP
const client = new MongoClient(
  'mongodb://192.168.1.10:27017,192.168.1.11:27017,192.168.1.12:27017/db?replicaSet=rs0'
);
```

#### Solution 5 : Configurer des DNS secondaires

```bash
# /etc/resolv.conf
nameserver 8.8.8.8      # Google DNS primaire
nameserver 8.8.4.4      # Google DNS secondaire
nameserver 1.1.1.1      # Cloudflare DNS

# Options
options timeout:2       # Timeout de 2 secondes
options attempts:3      # 3 tentatives
options rotate          # Alterner entre les serveurs DNS
```

---

## 7. Problèmes de Configuration

### Symptômes

```
Error parsing YAML config file
Configuration file option 'X' is not recognized
Unexpected argument: X
```

### Causes Possibles

- Fichier de configuration mal formaté (YAML invalide)
- Options obsolètes ou non supportées
- Conflit entre ligne de commande et fichier config
- Permissions incorrectes sur le fichier

---

### Diagnostic Pas à Pas

#### Étape 1 : Valider la syntaxe YAML

```bash
# Vérifier la syntaxe avec un parser YAML
python3 -c "import yaml; yaml.safe_load(open('/etc/mongod.conf'))"

# Ou avec yamllint
sudo apt-get install yamllint
yamllint /etc/mongod.conf

# Afficher la configuration parsée
cat /etc/mongod.conf
```

#### Étape 2 : Vérifier la configuration actuelle

```bash
# Afficher la configuration complète
mongod --config /etc/mongod.conf --print-config

# Voir les options appliquées
mongosh --eval "db.adminCommand({getCmdLineOpts: 1})"
```

#### Étape 3 : Tester le démarrage en mode verbose

```bash
# Arrêter MongoDB
sudo systemctl stop mongod

# Démarrer en manuel avec logs verbeux
sudo -u mongodb mongod --config /etc/mongod.conf --verbose

# Observer les messages d'erreur détaillés
```

#### Étape 4 : Comparer avec une configuration par défaut

```bash
# Sauvegarder l'actuelle
sudo cp /etc/mongod.conf /etc/mongod.conf.backup

# Télécharger une configuration de référence
wget https://raw.githubusercontent.com/mongodb/mongo/master/debian/mongod.conf -O /tmp/mongod.conf.default

# Comparer
diff /etc/mongod.conf /tmp/mongod.conf.default
```

---

### Résolution Pas à Pas

#### Solution 1 : Corriger les erreurs de syntaxe YAML

**Erreurs communes :**

```yaml
# ❌ INCORRECT : Mauvaise indentation
net:
port: 27017  # Devrait être indenté

# ✅ CORRECT
net:
  port: 27017

# ❌ INCORRECT : Espaces dans la valeur sans quotes
storage:
  dbPath: /var/lib/mongodb data  # Espace sans quotes

# ✅ CORRECT
storage:
  dbPath: "/var/lib/mongodb data"

# ❌ INCORRECT : Mélange tabs et espaces
net:
	port: 27017  # Tab utilisé

# ✅ CORRECT : Toujours utiliser des espaces
net:
  port: 27017
```

**Configuration valide complète :**

```yaml
# /etc/mongod.conf

# Système
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  verbosity: 0

# Stockage
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1

# Réseau
net:
  port: 27017
  bindIp: 127.0.0.1
  maxIncomingConnections: 65536

# Processus
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

# Sécurité
security:
  authorization: enabled

# Réplication
replication:
  replSetName: rs0
```

#### Solution 2 : Migrer les options obsolètes

```yaml
# MongoDB 4.x → 5.x+ : ssl → tls

# ❌ OBSOLÈTE
net:
  ssl:
    mode: requireSSL
    PEMKeyFile: /path/to/cert.pem

# ✅ MODERNE
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /path/to/cert.pem
```

#### Solution 3 : Résoudre les conflits de configuration

```bash
# Voir toutes les sources de configuration
mongod --help

# Priorité : Ligne de commande > Fichier config > Valeurs par défaut

# Si démarrage via systemd
sudo nano /lib/systemd/system/mongod.service

# Vérifier ExecStart
# Ne pas mélanger --config et options individuelles
ExecStart=/usr/bin/mongod --config /etc/mongod.conf

# Pas de :
# ExecStart=/usr/bin/mongod --config /etc/mongod.conf --port 27018
```

#### Solution 4 : Corriger les permissions

```bash
# Le fichier de configuration doit être lisible
sudo chmod 644 /etc/mongod.conf
sudo chown root:root /etc/mongod.conf

# Les répertoires doivent appartenir à mongodb
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb

# Le répertoire de PID
sudo mkdir -p /var/run/mongodb
sudo chown -R mongodb:mongodb /var/run/mongodb
```

#### Solution 5 : Valider avant redémarrage

```bash
# Tester la configuration avant d'appliquer
sudo mongod --config /etc/mongod.conf --validate

# Ou tester manuellement
sudo -u mongodb mongod --config /etc/mongod.conf --nojournal --bind_ip 127.0.0.1 --port 27999

# Si ça fonctionne, CTRL+C et redémarrer normalement
sudo systemctl restart mongod
```

---

## 8. Checklist Globale de Diagnostic

### Problème de Connexion : Par Où Commencer ?

```
┌─────────────────────────────────────────┐
│  MongoDB accessible localement ?        │
└──────────────┬──────────────────────────┘
               │
       ┌───────┴───────┐
       │  OUI          │  NON
       │               │
       ▼               ▼
  ┌─────────┐    ┌──────────────┐
  │ Réseau/ │    │ MongoDB      │
  │ Firewall│    │ pas démarré  │
  │ DNS     │    │ ou crash     │
  └─────────┘    └──────────────┘
```

### Checklist Rapide (5 minutes)

```bash
# 1. MongoDB est-il démarré ?
sudo systemctl status mongod

# 2. Écoute-t-il sur le bon port ?
netstat -tuln | grep 27017

# 3. Logs récents ?
tail -n 50 /var/log/mongodb/mongod.log

# 4. Connexion locale fonctionne ?
mongosh --host 127.0.0.1

# 5. DNS fonctionne ?
nslookup <mongodb_host>

# 6. Port accessible depuis client ?
telnet <mongodb_host> 27017

# 7. Authentification correcte ?
mongosh -u admin -p --authenticationDatabase admin

# 8. Certificats valides ?
openssl x509 -in /path/to/cert.pem -noout -dates
```

### Checklist Approfondie (30 minutes)

#### Système

- [ ] MongoDB démarre correctement
- [ ] Pas d'erreurs dans les logs système (`journalctl -u mongod`)
- [ ] Espace disque disponible (`df -h`)
- [ ] Mémoire disponible (`free -h`)
- [ ] Limites système correctes (`ulimit -a`)
- [ ] Permissions correctes sur /var/lib/mongodb

#### Réseau

- [ ] bindIp configuré correctement
- [ ] Port 27017 ouvert dans le pare-feu
- [ ] DNS résout correctement les noms d'hôtes
- [ ] Latence réseau acceptable (`ping`, `mtr`)
- [ ] Pas de problèmes de routage
- [ ] TCP keepalive configuré

#### Authentification

- [ ] Utilisateur existe
- [ ] Mot de passe correct
- [ ] Base d'authentification correcte (`authSource`)
- [ ] Rôles et privilèges suffisants
- [ ] Mécanisme d'authentification supporté

#### TLS/SSL

- [ ] Certificats non expirés
- [ ] Certificats valides (chaîne complète)
- [ ] CN/SAN correspond aux hostnames
- [ ] CA reconnu par le client
- [ ] Configuration TLS/SSL cohérente (client/serveur)

#### Pool de Connexions

- [ ] Pool size correctement dimensionné
- [ ] Pas de fuites de connexions
- [ ] Connexions fermées proprement
- [ ] Timeouts appropriés configurés
- [ ] Limite maxIncomingConnections suffisante

---

## Scripts de Diagnostic Automatisés

### Script de Diagnostic Complet

```bash
#!/bin/bash
# mongodb-connection-diagnostic.sh

echo "=== MongoDB Connection Diagnostic Tool ==="
echo "Date: $(date)"
echo ""

# Variables
MONGO_HOST=${1:-localhost}
MONGO_PORT=${2:-27017}
LOG_FILE="/var/log/mongodb/mongod.log"

# 1. MongoDB Service
echo "1. Checking MongoDB Service..."
systemctl is-active mongod && echo "✓ Service is running" || echo "✗ Service is NOT running"
echo ""

# 2. Process
echo "2. Checking MongoDB Process..."
ps aux | grep -v grep | grep mongod && echo "✓ Process found" || echo "✗ Process NOT found"
echo ""

# 3. Network
echo "3. Checking Network Binding..."
netstat -tuln | grep :$MONGO_PORT && echo "✓ Listening on port $MONGO_PORT" || echo "✗ NOT listening"
echo ""

# 4. DNS
echo "4. Testing DNS Resolution..."
nslookup $MONGO_HOST > /dev/null 2>&1 && echo "✓ DNS resolves" || echo "✗ DNS resolution failed"
echo ""

# 5. Port Connectivity
echo "5. Testing Port Connectivity..."
timeout 2 bash -c "cat < /dev/null > /dev/tcp/$MONGO_HOST/$MONGO_PORT" 2>/dev/null && \
  echo "✓ Port $MONGO_PORT accessible" || echo "✗ Port $MONGO_PORT NOT accessible"
echo ""

# 6. Disk Space
echo "6. Checking Disk Space..."
df -h /var/lib/mongodb
echo ""

# 7. Memory
echo "7. Checking Memory..."
free -h
echo ""

# 8. Recent Logs
echo "8. Recent Errors in Logs..."
if [ -f "$LOG_FILE" ]; then
  grep -i "error" $LOG_FILE | tail -n 5
else
  echo "✗ Log file not found"
fi
echo ""

# 9. Connection Test
echo "9. Testing Local Connection..."
mongosh --host $MONGO_HOST --port $MONGO_PORT --eval "db.adminCommand({ping: 1})" > /dev/null 2>&1 && \
  echo "✓ Connection successful" || echo "✗ Connection failed"
echo ""

echo "=== Diagnostic Complete ==="
```

**Utilisation :**

```bash
chmod +x mongodb-connection-diagnostic.sh
sudo ./mongodb-connection-diagnostic.sh
# Ou pour un hôte distant
sudo ./mongodb-connection-diagnostic.sh mongodb.example.com 27017
```

---

## Procédures d'Urgence

### Procédure : MongoDB Complètement Inaccessible

```
INCIDENT CRITIQUE : MongoDB totalement inaccessible

ÉTAPE 1 : NE PAS PANIQUER
- Respirer
- Rassembler l'équipe
- Démarrer le timer

ÉTAPE 2 : TRIAGE RAPIDE (2 minutes)
□ Service actif ? → systemctl status mongod
□ Process actif ? → ps aux | grep mongod
□ Port ouvert ? → netstat -tuln | grep 27017

ÉTAPE 3 : ACTIONS SELON SCENARIO

Si service down :
  → systemctl start mongod
  → Vérifier logs : tail -f /var/log/mongodb/mongod.log
  → Si échec, voir logs détaillés

Si service up mais pas de connexion :
  → Vérifier bindIp
  → Vérifier pare-feu
  → Tester connexion locale

Si authentication fail :
  → Vérifier credentials
  → Voir logs d'auth
  → Envisager mode noauth temporaire si critique

ÉTAPE 4 : ESCALADE (si non résolu en 15 min)
  → Appeler l'expert MongoDB de garde
  → Préparer diagnostic complet
  → Considérer le failover vers secondary

ÉTAPE 5 : COMMUNICATION
  → Notifier les stakeholders
  → Donner ETR (Estimated Time to Resolution)
  → Updates toutes les 15 minutes
```

---

## Bonnes Pratiques de Prévention

### Monitoring Proactif

```javascript
// Script de monitoring des connexions
function monitorConnections() {
  const stats = db.serverStatus().connections;
  const threshold = 0.8; // 80% de la capacité

  const usage = stats.current / (stats.current + stats.available);

  if (usage > threshold) {
    // Alerte
    console.error(`WARNING: Connection usage at ${(usage * 100).toFixed(1)}%`);
    // Envoyer notification (email, Slack, PagerDuty)
  }
}

// Exécuter toutes les minutes
setInterval(monitorConnections, 60000);
```

### Documentation des Configurations

```bash
# Maintenir un fichier de documentation
/etc/mongodb/
├── mongod.conf              # Config active
├── mongod.conf.backup       # Backup de la dernière config stable
├── mongod.conf.template     # Template pour nouveaux déploiements
└── README.md                # Documentation des changements

# Historique des changements
git init /etc/mongodb
git add mongod.conf
git commit -m "Initial configuration"
```

### Tests Réguliers

```bash
# Script de test de connexion (à exécuter dans un cron)
#!/bin/bash
# test-mongodb-connection.sh

mongosh --host mongodb.local --eval "db.adminCommand({ping: 1})" > /dev/null 2>&1

if [ $? -ne 0 ]; then
  echo "ALERT: MongoDB connection test failed" | mail -s "MongoDB Alert" admin@example.com
  # Envoyer vers système de monitoring
fi
```

---

## Conclusion

Les problèmes de connexion MongoDB peuvent avoir de multiples causes, mais avec une approche méthodique :

1. **Identifier rapidement** la catégorie du problème
2. **Utiliser les outils** de diagnostic appropriés
3. **Appliquer les solutions** pas à pas
4. **Documenter** l'incident et la résolution
5. **Mettre en place** des mesures préventives

**Points clés à retenir :**

- ✅ Toujours vérifier les logs en premier
- ✅ Tester la connexion locale avant d'investiguer le réseau
- ✅ Une seule modification à la fois
- ✅ Documenter chaque action
- ✅ Ne jamais désactiver la sécurité en production sans processus établi

---


⏭️ [Problèmes de performance](/22-depannage-resolution-problemes/02-problemes-performance.md)
