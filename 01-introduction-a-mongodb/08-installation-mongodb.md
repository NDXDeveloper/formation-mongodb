🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.8 Installation de MongoDB (Windows, Linux, macOS)

## Introduction

Cette section vous guide pas à pas dans l'installation de MongoDB sur votre système d'exploitation. Nous couvrirons les trois principales plateformes : Windows, Linux (Ubuntu/Debian et CentOS/RHEL) et macOS.

À la fin de cette section, vous aurez une instance MongoDB fonctionnelle sur votre machine, prête pour le développement.

---

## Prérequis généraux

Avant de commencer l'installation, vérifiez que votre système répond aux exigences minimales :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Configuration minimale requise                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Processeur    : x86_64 (64 bits obligatoire)                      │
│   RAM           : 4 Go minimum (8 Go recommandé)                    │
│   Stockage      : 10 Go minimum pour les données                    │
│   Système       : 64 bits uniquement                                │
│                                                                     │
│   Versions supportées :                                             │
│   • Windows 10/11, Windows Server 2016+                             │
│   • Ubuntu 20.04, 22.04, 24.04 LTS                                  │
│   • Debian 11, 12                                                   │
│   • RHEL/CentOS 7, 8, 9                                             │
│   • macOS 11 (Big Sur) et versions ultérieures                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Choix de la version

MongoDB propose deux éditions :

| Édition | Description | Usage |
|---------|-------------|-------|
| **Community** | Gratuite, open source | Développement, petites productions |
| **Enterprise** | Payante, fonctionnalités avancées | Grandes entreprises, support officiel |

> **Note** : Ce tutoriel utilise l'édition **Community**, gratuite et suffisante pour l'apprentissage et la plupart des projets.

---

## Installation sur Windows

### Méthode 1 : Installateur graphique (recommandée pour débutants)

#### Étape 1 : Télécharger l'installateur

1. Rendez-vous sur le site officiel : [https://www.mongodb.com/try/download/community](https://www.mongodb.com/try/download/community)

2. Sélectionnez les options suivantes :
   - **Version** : La plus récente (8.x recommandé)
   - **Platform** : Windows
   - **Package** : msi

3. Cliquez sur **Download**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Page de téléchargement                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   MongoDB Community Server Download                                 │
│                                                                     │
│   Version:    [8.0.x (current)    ▼]                                │
│   Platform:   [Windows            ▼]                                │
│   Package:    [msi                ▼]                                │
│                                                                     │
│   [     Download     ]                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Étape 2 : Exécuter l'installateur

1. Double-cliquez sur le fichier `.msi` téléchargé
2. Acceptez les conditions d'utilisation
3. Choisissez **Complete** pour une installation complète

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Type d'installation                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ○ Complete (Recommandé)                                           │
│     Installe tous les composants                                    │
│     Emplacement : C:\Program Files\MongoDB\Server\8.0\              │
│                                                                     │
│   ○ Custom                                                          │
│     Permet de choisir les composants et l'emplacement               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Étape 3 : Configuration du service

L'installateur propose de configurer MongoDB en tant que service Windows :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Configuration du service                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ☑ Install MongoDB as a Service                                   │
│                                                                     │
│   ○ Run service as Network Service user (recommandé)                │
│   ○ Run service as a local or domain user                           │
│                                                                     │
│   Service Name: MongoDB                                             │
│                                                                     │
│   Data Directory:  C:\Program Files\MongoDB\Server\8.0\data\        │
│   Log Directory:   C:\Program Files\MongoDB\Server\8.0\log\         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

> **Recommandation** : Cochez "Install MongoDB as a Service" pour que MongoDB démarre automatiquement avec Windows.

#### Étape 4 : MongoDB Compass (optionnel)

L'installateur propose d'installer **MongoDB Compass**, l'interface graphique officielle :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MongoDB Compass                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ☑ Install MongoDB Compass                                        │
│                                                                     │
│   MongoDB Compass est une interface graphique pour :                │
│   • Explorer vos données visuellement                               │
│   • Exécuter des requêtes                                           │
│   • Analyser les performances                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

> **Conseil** : Installez Compass, c'est un excellent outil pour les débutants.

#### Étape 5 : Terminer l'installation

Cliquez sur **Install** puis **Finish** une fois l'installation terminée.

#### Étape 6 : Ajouter MongoDB au PATH (optionnel mais recommandé)

Pour utiliser les commandes MongoDB depuis n'importe quel terminal :

1. Ouvrez les **Paramètres système avancés**
   - Clic droit sur "Ce PC" → Propriétés → Paramètres système avancés

2. Cliquez sur **Variables d'environnement**

3. Dans "Variables système", sélectionnez **Path** et cliquez sur **Modifier**

4. Ajoutez le chemin : `C:\Program Files\MongoDB\Server\8.0\bin`

5. Validez avec **OK**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Variable PATH                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   C:\Windows\system32                                               │
│   C:\Windows                                                        │
│   C:\Program Files\MongoDB\Server\8.0\bin    ← Ajouter cette ligne  │
│   ...                                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Étape 7 : Vérifier l'installation

Ouvrez un nouveau terminal (PowerShell ou CMD) et tapez :

```powershell
# Vérifier la version de MongoDB
mongod --version

# Vérifier la version du shell
mongosh --version
```

Résultat attendu :

```
db version v8.0.x
Build Info: {
    "version": "8.0.x",
    ...
}
```

### Méthode 2 : Installation via winget (Windows Package Manager)

Si vous avez Windows 10/11 avec winget installé :

```powershell
# Installer MongoDB Community Server
winget install MongoDB.Server

# Installer MongoDB Shell
winget install MongoDB.Shell

# Installer MongoDB Compass
winget install MongoDB.Compass
```

### Gestion du service Windows

```powershell
# Vérifier le statut du service
Get-Service MongoDB

# Démarrer le service
Start-Service MongoDB

# Arrêter le service
Stop-Service MongoDB

# Redémarrer le service
Restart-Service MongoDB
```

Ou via l'interface graphique :
1. Ouvrez **services.msc**
2. Trouvez **MongoDB**
3. Clic droit pour démarrer/arrêter

---

## Installation sur Linux

### Ubuntu / Debian

#### Étape 1 : Importer la clé GPG publique

```bash
# Installer les prérequis
sudo apt-get install -y gnupg curl

# Importer la clé GPG de MongoDB
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
   --dearmor
```

#### Étape 2 : Ajouter le dépôt MongoDB

**Pour Ubuntu 22.04 (Jammy) :**

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

**Pour Ubuntu 24.04 (Noble) :**

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

**Pour Debian 12 (Bookworm) :**

```bash
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] http://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

#### Étape 3 : Installer MongoDB

```bash
# Mettre à jour la liste des paquets
sudo apt-get update

# Installer MongoDB
sudo apt-get install -y mongodb-org
```

Cette commande installe les paquets suivants :

| Paquet | Description |
|--------|-------------|
| `mongodb-org` | Méta-paquet qui installe tout |
| `mongodb-org-server` | Le démon mongod |
| `mongodb-org-mongos` | Le routeur mongos |
| `mongodb-org-shell` | Le shell mongosh |
| `mongodb-org-tools` | Outils (mongodump, mongorestore, etc.) |

#### Étape 4 : Démarrer MongoDB

```bash
# Démarrer le service
sudo systemctl start mongod

# Activer le démarrage automatique
sudo systemctl enable mongod

# Vérifier le statut
sudo systemctl status mongod
```

Résultat attendu :

```
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; enabled)
     Active: active (running) since ...
```

#### Étape 5 : Vérifier l'installation

```bash
# Vérifier la version
mongod --version

# Se connecter au shell
mongosh
```

Dans le shell MongoDB :

```javascript
// Afficher la version du serveur
db.version()

// Tester une commande simple
db.adminCommand({ ping: 1 })
```

### CentOS / RHEL / Fedora

#### Étape 1 : Créer le fichier de dépôt

```bash
# Créer le fichier de configuration du dépôt
sudo tee /etc/yum.repos.d/mongodb-org-8.0.repo << 'EOF'
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-8.0.asc
EOF
```

#### Étape 2 : Installer MongoDB

```bash
# Installer MongoDB
sudo yum install -y mongodb-org

# Ou avec dnf (RHEL 8+, Fedora)
sudo dnf install -y mongodb-org
```

#### Étape 3 : Démarrer MongoDB

```bash
# Démarrer le service
sudo systemctl start mongod

# Activer le démarrage automatique
sudo systemctl enable mongod

# Vérifier le statut
sudo systemctl status mongod
```

#### Configuration SELinux (si activé)

Si SELinux est activé et que MongoDB ne démarre pas :

```bash
# Option 1 : Autoriser MongoDB
sudo semanage port -a -t mongod_port_t -p tcp 27017

# Option 2 : Mettre SELinux en mode permissif (moins sécurisé)
sudo setenforce 0
```

### Structure des répertoires (Linux)

```
┌─────────────────────────────────────────────────────────────────────┐
│              Répertoires MongoDB sur Linux                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   /var/lib/mongodb/          → Données (dbPath)                     │
│   /var/log/mongodb/          → Logs                                 │
│   /etc/mongod.conf           → Fichier de configuration             │
│   /usr/bin/mongod            → Exécutable serveur                   │
│   /usr/bin/mongosh           → Exécutable shell                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Commandes de gestion du service (Linux)

```bash
# Démarrer MongoDB
sudo systemctl start mongod

# Arrêter MongoDB
sudo systemctl stop mongod

# Redémarrer MongoDB
sudo systemctl restart mongod

# Recharger la configuration
sudo systemctl reload mongod

# Voir les logs
sudo journalctl -u mongod

# Voir les logs en temps réel
sudo tail -f /var/log/mongodb/mongod.log
```

---

## Installation sur macOS

### Méthode 1 : Homebrew (recommandée)

#### Étape 1 : Installer Homebrew (si pas déjà installé)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### Étape 2 : Ajouter le tap MongoDB

```bash
brew tap mongodb/brew
```

#### Étape 3 : Installer MongoDB

```bash
# Installer MongoDB Community Edition
brew install mongodb-community@8.0
```

Cette commande installe :
- Le serveur `mongod`
- Le shell `mongosh`
- Les outils de base de données

#### Étape 4 : Démarrer MongoDB

```bash
# Démarrer MongoDB en tant que service
brew services start mongodb-community@8.0

# Ou démarrer manuellement (premier plan)
mongod --config /opt/homebrew/etc/mongod.conf
```

#### Étape 5 : Vérifier l'installation

```bash
# Vérifier que le service tourne
brew services list

# Vérifier la version
mongod --version

# Se connecter au shell
mongosh
```

### Structure des répertoires (macOS avec Homebrew)

```
┌─────────────────────────────────────────────────────────────────────┐
│              Répertoires MongoDB sur macOS (Apple Silicon)          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   /opt/homebrew/var/mongodb/           → Données (dbPath)           │
│   /opt/homebrew/var/log/mongodb/       → Logs                       │
│   /opt/homebrew/etc/mongod.conf        → Configuration              │
│   /opt/homebrew/bin/mongod             → Exécutable serveur         │
│   /opt/homebrew/bin/mongosh            → Exécutable shell           │
│                                                                     │
│   Note : Sur Intel Mac, remplacez /opt/homebrew par /usr/local      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Commandes de gestion du service (macOS)

```bash
# Démarrer MongoDB
brew services start mongodb-community@8.0

# Arrêter MongoDB
brew services stop mongodb-community@8.0

# Redémarrer MongoDB
brew services restart mongodb-community@8.0

# Voir le statut des services
brew services list

# Voir les logs
cat /opt/homebrew/var/log/mongodb/mongo.log

# Ou sur Intel Mac
cat /usr/local/var/log/mongodb/mongo.log
```

### Méthode 2 : Téléchargement manuel

1. Téléchargez le fichier `.tgz` depuis [mongodb.com/try/download/community](https://www.mongodb.com/try/download/community)

2. Extrayez l'archive :

```bash
tar -zxvf mongodb-macos-*.tgz
```

3. Déplacez les binaires :

```bash
sudo cp mongodb-macos-*/bin/* /usr/local/bin/
```

4. Créez les répertoires nécessaires :

```bash
sudo mkdir -p /usr/local/var/mongodb
sudo mkdir -p /usr/local/var/log/mongodb
```

5. Définissez les permissions :

```bash
sudo chown $(whoami) /usr/local/var/mongodb
sudo chown $(whoami) /usr/local/var/log/mongodb
```

6. Démarrez MongoDB :

```bash
mongod --dbpath /usr/local/var/mongodb --logpath /usr/local/var/log/mongodb/mongo.log --fork
```

---

## Configuration de base

### Le fichier de configuration

MongoDB utilise un fichier de configuration au format YAML. Voici les emplacements par défaut :

| Système | Emplacement |
|---------|-------------|
| Windows | `C:\Program Files\MongoDB\Server\8.0\bin\mongod.cfg` |
| Linux | `/etc/mongod.conf` |
| macOS (Homebrew) | `/opt/homebrew/etc/mongod.conf` |

### Exemple de configuration de base

```yaml
# mongod.conf - Configuration MongoDB

# Stockage des données
storage:
  dbPath: /var/lib/mongodb        # Chemin des données
  journal:
    enabled: true                  # Journaling activé (recommandé)

# Logs
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true                  # Ajouter aux logs existants

# Réseau
net:
  port: 27017                      # Port d'écoute
  bindIp: 127.0.0.1               # Écouter uniquement en local

# Processus
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: true                       # Exécuter en arrière-plan (Linux/macOS)
```

### Options de configuration importantes

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Options de configuration clés                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   storage.dbPath                                                    │
│   └── Répertoire où MongoDB stocke les fichiers de données          │
│                                                                     │
│   net.port                                                          │
│   └── Port TCP d'écoute (défaut: 27017)                             │
│                                                                     │
│   net.bindIp                                                        │
│   └── Adresses IP sur lesquelles écouter                            │
│       • 127.0.0.1 = localhost uniquement (sécurisé)                 │
│       • 0.0.0.0 = toutes les interfaces (attention !)               │
│                                                                     │
│   security.authorization                                            │
│   └── Activer l'authentification (enabled/disabled)                 │
│                                                                     │
│   storage.wiredTiger.engineConfig.cacheSizeGB                       │
│   └── Taille du cache WiredTiger en mémoire                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Appliquer les changements de configuration

Après avoir modifié le fichier de configuration, redémarrez MongoDB :

```bash
# Linux
sudo systemctl restart mongod

# macOS
brew services restart mongodb-community@8.0

# Windows (PowerShell admin)
Restart-Service MongoDB
```

---

## Vérification de l'installation

### Test de connexion

```bash
# Se connecter au shell MongoDB
mongosh

# Ou avec une URI explicite
mongosh "mongodb://localhost:27017"
```

### Commandes de test dans mongosh

```javascript
// Afficher les bases de données
show dbs

// Créer une base de test
use testdb

// Insérer un document
db.test.insertOne({ message: "MongoDB fonctionne !", date: new Date() })

// Lire le document
db.test.findOne()

// Supprimer la base de test
db.dropDatabase()

// Quitter le shell
exit
```

### Résultat attendu

```
test> db.test.insertOne({ message: "MongoDB fonctionne !", date: new Date() })
{
  acknowledged: true,
  insertedId: ObjectId('507f1f77bcf86cd799439011')
}

test> db.test.findOne()
{
  _id: ObjectId('507f1f77bcf86cd799439011'),
  message: 'MongoDB fonctionne !',
  date: ISODate('2024-11-28T10:30:00.000Z')
}
```

### Vérifier les informations du serveur

```javascript
// Dans mongosh
db.serverStatus()

// Version du serveur
db.version()

// Informations sur l'hôte
db.hostInfo()

// Statistiques du serveur
db.serverStatus().connections
```

---

## Dépannage courant

### Problème 1 : MongoDB ne démarre pas

#### Symptôme
```
Failed to start mongod.service: Unit mongod.service not found.
```

#### Solutions

```bash
# Vérifier que MongoDB est installé
which mongod

# Recharger systemd (Linux)
sudo systemctl daemon-reload

# Vérifier les logs
sudo cat /var/log/mongodb/mongod.log
```

### Problème 2 : Erreur de permission sur le répertoire de données

#### Symptôme
```
exception in initAndListen: NonExistentPath: Data directory /var/lib/mongodb not found
```

#### Solutions

```bash
# Créer le répertoire
sudo mkdir -p /var/lib/mongodb

# Attribuer les bonnes permissions
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo chmod 755 /var/lib/mongodb
```

### Problème 3 : Port 27017 déjà utilisé

#### Symptôme
```
addr already in use
```

#### Solutions

```bash
# Trouver le processus qui utilise le port (Linux/macOS)
sudo lsof -i :27017

# Tuer le processus si nécessaire
sudo kill -9 <PID>

# Ou changer le port dans mongod.conf
net:
  port: 27018
```

### Problème 4 : Impossible de se connecter

#### Symptôme
```
MongoNetworkError: connect ECONNREFUSED 127.0.0.1:27017
```

#### Solutions

```bash
# Vérifier que MongoDB tourne
sudo systemctl status mongod     # Linux
brew services list               # macOS

# Vérifier que le port est ouvert
netstat -an | grep 27017

# Vérifier la configuration bindIp
cat /etc/mongod.conf | grep bindIp
```

### Problème 5 : Erreur de lock file

#### Symptôme
```
Unable to lock the lock file: /var/lib/mongodb/mongod.lock
```

#### Solutions

```bash
# Supprimer le fichier de lock (si MongoDB n'est pas en cours d'exécution)
sudo rm /var/lib/mongodb/mongod.lock

# Réparer la base de données
mongod --dbpath /var/lib/mongodb --repair

# Redémarrer
sudo systemctl start mongod
```

### Tableau récapitulatif du dépannage

| Problème | Cause probable | Solution |
|----------|----------------|----------|
| Service non trouvé | Installation incomplète | Réinstaller MongoDB |
| Permission denied | Mauvais propriétaire | `chown mongodb:mongodb` |
| Port utilisé | Autre instance | Tuer le processus ou changer le port |
| Connection refused | Service arrêté | Démarrer le service |
| Lock file | Arrêt brutal | Supprimer le lock, réparer |

---

## Récapitulatif des commandes par système

### Windows (PowerShell)

```powershell
# Statut du service
Get-Service MongoDB

# Démarrer
Start-Service MongoDB

# Arrêter
Stop-Service MongoDB

# Connexion
mongosh
```

### Linux (systemd)

```bash
# Statut du service
sudo systemctl status mongod

# Démarrer
sudo systemctl start mongod

# Arrêter
sudo systemctl stop mongod

# Activer au démarrage
sudo systemctl enable mongod

# Logs
sudo journalctl -u mongod -f

# Connexion
mongosh
```

### macOS (Homebrew)

```bash
# Statut du service
brew services list

# Démarrer
brew services start mongodb-community@8.0

# Arrêter
brew services stop mongodb-community@8.0

# Connexion
mongosh
```

---

## Conclusion

Vous avez maintenant MongoDB installé et fonctionnel sur votre système. Les points essentiels à retenir :

- MongoDB s'installe facilement via les gestionnaires de paquets (apt, yum, brew)
- Le service démarre sur le port **27017** par défaut
- Le fichier de configuration permet de personnaliser le comportement
- `mongosh` est le shell pour interagir avec MongoDB

Dans la prochaine section, nous verrons une méthode d'installation alternative et très populaire : **Docker**.

---

## Points clés à retenir

- MongoDB nécessite un système **64 bits**
- Le port par défaut est **27017**
- Fichiers de configuration :
  - Windows : `mongod.cfg`
  - Linux/macOS : `mongod.conf`
- Répertoire de données par défaut :
  - Windows : `C:\Program Files\MongoDB\Server\8.0\data\`
  - Linux : `/var/lib/mongodb/`
  - macOS : `/opt/homebrew/var/mongodb/`
- Utilisez `mongosh` pour vous connecter au serveur
- Activez le service au démarrage pour ne pas avoir à le lancer manuellement

---


⏭️ [Installation via Docker](/01-introduction-a-mongodb/09-installation-docker.md)
