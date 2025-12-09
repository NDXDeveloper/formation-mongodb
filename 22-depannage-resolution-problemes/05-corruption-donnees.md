🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.5 Corruption de Données

## Vue d'ensemble

La corruption de données est l'un des incidents les plus graves pouvant affecter MongoDB. Elle peut entraîner une perte de données, une indisponibilité du service et nécessite des procédures de récupération complexes. Cette section fournit des méthodologies complètes pour détecter, diagnostiquer et récupérer d'une corruption de données.

---

## Table des Matières

1. [Types de Corruption](#1-types-de-corruption)
2. [Détection de Corruption](#2-d%C3%A9tection-de-corruption)
3. [Corruption de Fichiers](#3-corruption-de-fichiers)
4. [Corruption d'Index](#4-corruption-dindex)
5. [Corruption de Documents](#5-corruption-de-documents)
6. [Corruption de Metadata](#6-corruption-de-metadata)
7. [Procédures de Récupération](#7-proc%C3%A9dures-de-r%C3%A9cup%C3%A9ration)
8. [Prévention](#8-pr%C3%A9vention)

---

## 1. Types de Corruption

### Classification de la Corruption

#### 1.1 Corruption au Niveau Fichier

**Causes :**
- Défaillance matérielle (disque dur)
- Arrêt brutal du système
- Problèmes de système de fichiers
- Erreurs de contrôleur RAID
- Corruption mémoire

**Symptômes :**
```
WiredTiger: WT_PANIC: WiredTiger library panic
File corruption detected
Checksum mismatch
Cannot read from file
Block corruption detected
```

#### 1.2 Corruption d'Index

**Causes :**
- Interruption pendant un build d'index
- Bug dans le code d'index
- Modifications concurrentes
- Problèmes de mémoire

**Symptômes :**
```
Index validation failed
Inconsistent index entries
Missing index keys
Duplicate keys in unique index
```

#### 1.3 Corruption de Documents

**Causes :**
- Écriture partielle
- Problèmes de réplication
- Bug applicatif
- Corruption BSON

**Symptômes :**
```
Invalid BSON detected
Document structure corrupted
Field type mismatch
Unable to parse document
```

#### 1.4 Corruption de Metadata

**Causes :**
- Opérations DDL interrompues
- Problèmes de config server (sharding)
- Corruption du catalog

**Symptômes :**
```
Collection metadata corrupted
Namespace not found
Catalog inconsistency
Invalid collection options
```

---

## 2. Détection de Corruption

### Méthodes de Détection

#### Étape 1 : Surveillance des Logs

```bash
# Rechercher les signes de corruption
grep -i "corrupt\|panic\|checksum\|error" /var/log/mongodb/mongod.log

# Messages critiques à surveiller
grep -E "WiredTiger.*panic|Fatal Assertion|WT_PANIC" /var/log/mongodb/mongod.log

# Erreurs de lecture/écriture
grep -E "error.*reading|error.*writing|I/O error" /var/log/mongodb/mongod.log

# Problèmes de validation
grep -i "validation\|invalid" /var/log/mongodb/mongod.log
```

**Messages critiques indiquant une corruption :**

```
# Corruption WiredTiger
[timestamp] F STORAGE  [conn123] WiredTiger error (0) [1234567890:123456][12345:0x123456789abc], file:collection-1--234567890.wt, WT_SESSION.checkpoint: __wt_block_read_off, 123: collection-1--234567890.wt: read checksum error

# Corruption de fichier
[timestamp] E STORAGE  [conn456] WiredTiger error (WT_ERROR) [1234567890:123456][12345:0x123456789abc], file:index-2--345678901.wt, WT_SESSION.open_cursor: __wt_block_read_off, 456: index-2--345678901.wt: fatal file corruption detected

# Panique WiredTiger
[timestamp] F -        [conn789] Fatal Assertion 50853 at src/mongo/db/storage/wiredtiger/wiredtiger_util.cpp 123
```

#### Étape 2 : Validation de Collection

```javascript
// Validation basique
db.mycollection.validate()

// Validation complète (plus lente mais exhaustive)
db.mycollection.validate({full: true})

// Sortie normale (pas de corruption)
{
  "ns": "mydb.mycollection",
  "nInvalidDocuments": 0,
  "nrecords": 1000000,
  "nIndexes": 3,
  "keysPerIndex": {
    "mydb.mycollection.$_id_": 1000000,
    "mydb.mycollection.$email_1": 1000000,
    "mydb.mycollection.$age_1": 1000000
  },
  "indexDetails": {...},
  "valid": true,
  "repaired": false,
  "warnings": [],
  "errors": [],
  "ok": 1
}

// Sortie avec corruption détectée
{
  "ns": "mydb.mycollection",
  "nInvalidDocuments": 5,
  "valid": false,
  "errors": [
    "Index 'email_1' is missing keys for documents",
    "Document corruption detected at record id 123456",
    "Checksum mismatch in collection data"
  ],
  "warnings": [
    "Collection has inconsistent record count"
  ],
  "ok": 1
}
```

#### Étape 3 : Validation Automatisée de Toutes les Collections

```javascript
// Script de validation globale
function validateAllCollections(dbName) {
  var results = {
    timestamp: new Date(),
    database: dbName,
    collections: [],
    totalInvalid: 0,
    totalErrors: 0
  }

  var db = db.getSiblingDB(dbName)
  var collections = db.getCollectionNames()

  collections.forEach(collName => {
    print(`Validating ${dbName}.${collName}...`)

    try {
      var validation = db[collName].validate({full: true})

      var collResult = {
        name: collName,
        valid: validation.valid,
        errors: validation.errors || [],
        warnings: validation.warnings || [],
        nInvalidDocuments: validation.nInvalidDocuments || 0
      }

      if (!validation.valid) {
        results.totalInvalid++
        results.totalErrors += validation.errors.length
        print(`  ⚠️  INVALID: ${validation.errors.join(', ')}`)
      } else {
        print(`  ✓ Valid`)
      }

      results.collections.push(collResult)

    } catch (e) {
      print(`  ✗ ERROR: ${e.message}`)
      results.collections.push({
        name: collName,
        valid: false,
        error: e.message
      })
      results.totalInvalid++
    }
  })

  return results
}

// Exécuter sur toutes les bases
db.adminCommand({listDatabases: 1}).databases.forEach(dbInfo => {
  if (!['admin', 'local', 'config'].includes(dbInfo.name)) {
    print(`\n=== Validating database: ${dbInfo.name} ===`)
    var results = validateAllCollections(dbInfo.name)
    printjson(results)
  }
})
```

#### Étape 4 : Vérification de l'Intégrité des Fichiers

```bash
# Sur le système de fichiers

# 1. Vérifier l'intégrité du système de fichiers
sudo fsck -n /dev/sda1  # -n = no write (safe)

# 2. Vérifier les erreurs disque
sudo smartctl -a /dev/sda

# 3. Vérifier les checksums WiredTiger (si activés)
# Arrêter MongoDB d'abord
sudo systemctl stop mongod

# Vérifier avec wiredtiger verify
wt -h /var/lib/mongodb verify collection-0--123456789.wt

# 4. Vérifier les permissions
ls -la /var/lib/mongodb/
# Tout doit appartenir à mongodb:mongodb

# 5. Vérifier l'espace disque
df -h /var/lib/mongodb
```

#### Étape 5 : Diagnostic des Symptômes

```javascript
// Tester les opérations de base

// 1. Lecture simple
try {
  db.mycollection.findOne()
  print("✓ Read operation successful")
} catch (e) {
  print("✗ Read failed:", e.message)
}

// 2. Écriture simple
try {
  db.mycollection.insertOne({test: true, timestamp: new Date()})
  print("✓ Write operation successful")
} catch (e) {
  print("✗ Write failed:", e.message)
}

// 3. Comptage
try {
  var count = db.mycollection.countDocuments({})
  print("✓ Count successful:", count)
} catch (e) {
  print("✗ Count failed:", e.message)
}

// 4. Index scan
try {
  db.mycollection.find().hint({_id: 1}).limit(10).toArray()
  print("✓ Index scan successful")
} catch (e) {
  print("✗ Index scan failed:", e.message)
}

// 5. Aggregation
try {
  db.mycollection.aggregate([{$limit: 1}])
  print("✓ Aggregation successful")
} catch (e) {
  print("✗ Aggregation failed:", e.message)
}
```

---

## 3. Corruption de Fichiers

### Diagnostic Approfondi

#### Étape 1 : Identifier les Fichiers Corrompus

```bash
# Lister les fichiers de données
ls -lh /var/lib/mongodb/*.wt

# Vérifier chaque fichier WiredTiger
cd /var/lib/mongodb
for file in *.wt; do
  echo "Checking $file..."
  wt verify $file 2>&1 | tee -a /tmp/wt-verify.log
done

# Analyser les résultats
grep -i "error\|corrupt" /tmp/wt-verify.log
```

#### Étape 2 : Vérifier le Journal WiredTiger

```bash
# Le journal WiredTiger contient les transactions non encore écrites
ls -lh /var/lib/mongodb/journal/

# Vérifier l'intégrité du journal
wt -h /var/lib/mongodb printlog | less

# Rechercher les erreurs
wt -h /var/lib/mongodb printlog 2>&1 | grep -i error
```

---

### Résolution Pas à Pas

#### Solution 1 : Restauration depuis Backup

**C'est la méthode PRÉFÉRÉE et la plus sûre.**

```bash
# 1. Arrêter MongoDB
sudo systemctl stop mongod

# 2. Sauvegarder l'état corrompu (pour analyse)
sudo mv /var/lib/mongodb /var/lib/mongodb.corrupted.$(date +%Y%m%d_%H%M%S)

# 3. Créer un nouveau répertoire
sudo mkdir -p /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/lib/mongodb

# 4. Restaurer depuis le backup
# Option A : Backup physique (snapshot)
sudo rsync -av /backup/mongodb/latest/ /var/lib/mongodb/

# Option B : Backup logique (mongodump)
mongorestore --drop /backup/mongodb/dump/

# 5. Vérifier les permissions
sudo chown -R mongodb:mongodb /var/lib/mongodb

# 6. Redémarrer MongoDB
sudo systemctl start mongod

# 7. Vérifier
mongosh --eval "db.adminCommand({ping: 1})"
mongosh --eval "db.getSiblingDB('mydb').mycollection.countDocuments({})"
```

#### Solution 2 : Réparation avec --repair (Dernier Recours)

**⚠️ ATTENTION : Peut causer une perte de données !**

```bash
# 1. Arrêter MongoDB
sudo systemctl stop mongod

# 2. Sauvegarder l'état actuel
sudo cp -r /var/lib/mongodb /var/lib/mongodb.backup.$(date +%Y%m%d_%H%M%S)

# 3. Vérifier l'espace disque disponible
# IMPORTANT : Repair nécessite au moins 1.5x la taille des données
df -h /var/lib/mongodb

# 4. Lancer la réparation
sudo -u mongodb mongod --dbpath /var/lib/mongodb --repair

# Sortie attendue :
# [timestamp] I STORAGE  [initandlisten] repairDatabase mydb
# [timestamp] I STORAGE  [initandlisten] validate collection mydb.mycollection
# [timestamp] I INDEX    [initandlisten] building index...
# [timestamp] I STORAGE  [initandlisten] repairDatabase mydb finished

# 5. Vérifier les logs de réparation
tail -100 /var/log/mongodb/mongod.log

# 6. Redémarrer normalement
sudo systemctl start mongod

# 7. Valider les données
mongosh --eval "db.getSiblingDB('mydb').mycollection.validate({full: true})"
```

**Options de --repair :**

```bash
# Réparation avec options spécifiques
sudo -u mongodb mongod \
  --dbpath /var/lib/mongodb \
  --repair \
  --repairpath /mnt/tmp/mongodb-repair  # Utiliser un disque différent
```

#### Solution 3 : Récupération depuis Replica Set

```bash
# Si membre d'un replica set, resynchroniser depuis un secondary sain

# 1. Sur le membre corrompu, arrêter MongoDB
sudo systemctl stop mongod

# 2. Supprimer les données corrompues
sudo rm -rf /var/lib/mongodb/*
# Garder le fichier mongod.conf !

# 3. Redémarrer MongoDB
sudo systemctl start mongod

# 4. MongoDB va automatiquement effectuer une initial sync
# depuis un membre sain du replica set

# 5. Surveiller les logs
tail -f /var/log/mongodb/mongod.log | grep -i "sync"

# Progression attendue :
# [timestamp] I REPL     [replication-0] Starting initial sync
# [timestamp] I REPL     [replication-0] Cloning database: mydb
# [timestamp] I REPL     [replication-0] Collection: mydb.mycollection
# [timestamp] I REPL     [replication-0] Initial sync done

# 6. Vérifier l'état du replica set
mongosh --eval "rs.status()"
```

#### Solution 4 : Récupération Partielle (Salvage)

**Pour récupérer le maximum de données quand --repair échoue.**

```bash
# 1. Arrêter MongoDB
sudo systemctl stop mongod

# 2. Utiliser wt salvage sur chaque fichier corrompu
cd /var/lib/mongodb

# Identifier le fichier corrompu (exemple)
wt salvage collection-0--123456789.wt

# Pour tous les fichiers collection
for file in collection-*.wt; do
  echo "Salvaging $file..."
  wt salvage $file
done

# 3. Redémarrer MongoDB
sudo systemctl start mongod

# 4. Valider et reconstruire les index
mongosh <<EOF
use mydb
db.mycollection.validate({full: true})

// Reconstruire tous les index
db.mycollection.getIndexes().forEach(function(idx) {
  if (idx.name !== "_id_") {
    print("Rebuilding index: " + idx.name)
    db.mycollection.dropIndex(idx.name)
    db.mycollection.createIndex(idx.key, {name: idx.name})
  }
})
EOF
```

#### Solution 5 : Export/Import Sélectif

**Récupérer uniquement les données valides.**

```bash
# 1. Exporter les données valides
mongodump --db mydb --collection mycollection \
  --query '{}' \
  --out /backup/recovery/

# Note : mongodump ignorera les documents corrompus

# 2. Vérifier l'export
ls -lh /backup/recovery/mydb/

# 3. Arrêter MongoDB et nettoyer
sudo systemctl stop mongod
sudo rm -rf /var/lib/mongodb/*

# 4. Redémarrer avec base vide
sudo systemctl start mongod

# 5. Importer les données valides
mongorestore --db mydb --collection mycollection \
  /backup/recovery/mydb/mycollection.bson

# 6. Recréer les index
mongosh mydb --eval "db.mycollection.getIndexes()" > /tmp/indexes.json
# Analyser et recréer manuellement
```

---

## 4. Corruption d'Index

### Diagnostic

```javascript
// Détecter les index corrompus
db.mycollection.validate({full: true})

// Sortie typique avec index corrompu :
// {
//   "valid": false,
//   "errors": [
//     "Index 'email_1' is missing entries for documents",
//     "Index 'age_1' has incorrect key count"
//   ]
// }

// Comparer les counts
db.mycollection.countDocuments({})  // Documents
db.mycollection.countDocuments({email: {$exists: true}})  // Avec email

// Si différence significative = index corrompu
```

---

### Résolution Pas à Pas

#### Solution 1 : Reconstruire les Index

```javascript
// Méthode simple : Reconstruire tous les index

// 1. Lister les index actuels
var indexes = db.mycollection.getIndexes()
printjson(indexes)

// 2. Sauvegarder la définition
var indexDefs = []
db.mycollection.getIndexes().forEach(idx => {
  if (idx.name !== "_id_") {  // Ne pas supprimer l'index _id
    indexDefs.push({
      key: idx.key,
      name: idx.name,
      unique: idx.unique || false,
      sparse: idx.sparse || false,
      expireAfterSeconds: idx.expireAfterSeconds,
      // Autres options...
    })
  }
})

// 3. Supprimer les index (sauf _id)
db.mycollection.getIndexes().forEach(idx => {
  if (idx.name !== "_id_") {
    print("Dropping index:", idx.name)
    db.mycollection.dropIndex(idx.name)
  }
})

// 4. Recréer les index
indexDefs.forEach(idxDef => {
  print("Creating index:", idxDef.name)

  var options = {
    name: idxDef.name,
    background: true  // Ne pas bloquer les opérations
  }

  if (idxDef.unique) options.unique = true
  if (idxDef.sparse) options.sparse = true
  if (idxDef.expireAfterSeconds) options.expireAfterSeconds = idxDef.expireAfterSeconds

  db.mycollection.createIndex(idxDef.key, options)
})

// 5. Vérifier
db.mycollection.validate({full: true})
```

#### Solution 2 : Rebuild Index avec reIndex()

```javascript
// ⚠️ Bloque les écritures pendant l'opération

// Pour une collection spécifique
db.mycollection.reIndex()

// Sortie :
// {
//   "nIndexesWas": 5,
//   "nIndexes": 5,
//   "indexes": [
//     {"name": "_id_", ...},
//     {"name": "email_1", ...},
//     // ...
//   ],
//   "ok": 1
// }

// Pour toutes les collections d'une base
db.getCollectionNames().forEach(collName => {
  print("Reindexing:", collName)
  db[collName].reIndex()
})
```

#### Solution 3 : Rebuild Progressif (Production)

```javascript
// Pour minimiser l'impact en production

// 1. Créer un nouvel index avec un nom différent
db.mycollection.createIndex(
  {email: 1},
  {
    name: "email_1_new",
    background: true
  }
)

// 2. Vérifier que le nouvel index est correct
db.mycollection.validate({full: true})

// 3. Supprimer l'ancien index
db.mycollection.dropIndex("email_1")

// 4. Renommer le nouvel index (MongoDB 4.2+)
db.runCommand({
  collMod: "mycollection",
  index: {
    keyPattern: {email: 1},
    name: "email_1"
  }
})
```

---

## 5. Corruption de Documents

### Diagnostic

```javascript
// Rechercher des documents BSON invalides

// 1. Parcourir la collection
var invalidDocs = []
var cursor = db.mycollection.find().batchSize(100)

try {
  cursor.forEach(doc => {
    try {
      // Tenter de sérialiser le document
      BSON.serialize(doc)

      // Vérifier la structure
      if (!doc._id) {
        invalidDocs.push({doc: doc, reason: "Missing _id"})
      }
    } catch (e) {
      invalidDocs.push({
        _id: doc._id || "unknown",
        error: e.message
      })
    }
  })
} catch (e) {
  print("Error during scan:", e.message)
}

print("Invalid documents found:", invalidDocs.length)
printjson(invalidDocs)

// 2. Vérifier les types de champs
db.mycollection.aggregate([
  {$project: {
    _id: 1,
    emailType: {$type: "$email"},
    ageType: {$type: "$age"}
  }},
  {$match: {
    $or: [
      {emailType: {$ne: "string"}},
      {ageType: {$nin: ["int", "long", "double"]}}
    ]
  }}
])
```

---

### Résolution Pas à Pas

#### Solution 1 : Supprimer les Documents Invalides

```javascript
// ⚠️ Perte de données - à utiliser avec précaution

// 1. Identifier et sauvegarder les documents invalides
var invalidDocs = []

db.mycollection.find().forEach(doc => {
  try {
    // Vérification basique
    if (!doc._id || typeof doc._id === 'undefined') {
      invalidDocs.push(doc)
    }

    // Vérifier la sérialisation BSON
    var serialized = BSON.serialize(doc)
    var deserialized = BSON.deserialize(serialized)

  } catch (e) {
    invalidDocs.push({
      _id: doc._id,
      error: e.message,
      doc: doc
    })
  }
})

// 2. Sauvegarder dans une collection séparée
invalidDocs.forEach(doc => {
  db.mycollection_invalid.insertOne(doc)
})

// 3. Supprimer de la collection principale
invalidDocs.forEach(doc => {
  if (doc._id) {
    db.mycollection.deleteOne({_id: doc._id})
  }
})

print("Removed " + invalidDocs.length + " invalid documents")
```

#### Solution 2 : Réparer les Documents

```javascript
// Corriger les problèmes de type de champs

// Exemple : Corriger les âges en string → number
db.mycollection.find({
  age: {$type: "string"}
}).forEach(doc => {
  db.mycollection.updateOne(
    {_id: doc._id},
    {$set: {age: parseInt(doc.age)}}
  )
})

// Exemple : Ajouter les champs manquants
db.mycollection.updateMany(
  {email: {$exists: false}},
  {$set: {email: null}}
)

// Exemple : Normaliser les formats de date
db.mycollection.find({
  created: {$type: "string"}
}).forEach(doc => {
  try {
    var date = new Date(doc.created)
    db.mycollection.updateOne(
      {_id: doc._id},
      {$set: {created: date}}
    )
  } catch (e) {
    print("Cannot parse date for doc:", doc._id)
  }
})
```

---

## 6. Corruption de Metadata

### Diagnostic

```javascript
// Vérifier l'intégrité du catalog

// 1. Lister les collections
db.getCollectionNames()

// Si erreur : "unable to list collections" = corruption metadata

// 2. Vérifier les statistics
db.stats()

// 3. Vérifier une collection spécifique
db.getCollectionInfos({name: "mycollection"})

// 4. Sur un cluster shardé, vérifier config database
use config
db.collections.find()
db.chunks.find()
db.shards.find()
```

---

### Résolution Pas à Pas

#### Solution 1 : Reconstruire le Catalog

```bash
# MongoDB 4.4+

# 1. Arrêter MongoDB
sudo systemctl stop mongod

# 2. Démarrer avec option de réparation du catalog
sudo -u mongodb mongod \
  --dbpath /var/lib/mongodb \
  --repair \
  --repairpath /mnt/tmp/repair

# 3. Redémarrer normalement
sudo systemctl start mongod
```

#### Solution 2 : Réinitialiser la Configuration Sharding

```javascript
// ⚠️ Cluster shardé uniquement

// Sur le config server

// 1. Sauvegarder la configuration actuelle
mongoexport --db config --collection collections \
  --out /backup/config.collections.json

mongoexport --db config --collection chunks \
  --out /backup/config.chunks.json

// 2. Si corruption détectée, reconstruire depuis un backup
// Ou depuis un autre config server (si replica set)

// 3. Forcer une reconfiguration
sh.stopBalancer()
// Corriger les métadonnées manuellement
sh.startBalancer()
```

---

## 7. Procédures de Récupération

### Matrice de Décision

```
TYPE DE CORRUPTION          | SOLUTION PRÉFÉRÉE
----------------------------|-----------------------------------
Fichiers WiredTiger         | 1. Restore backup
                           | 2. Resync depuis replica set
                           | 3. --repair
                           | 4. wt salvage
----------------------------|-----------------------------------
Index corrompus            | 1. Rebuild index
                           | 2. reIndex()
----------------------------|-----------------------------------
Documents invalides        | 1. Export/import sélectif
                           | 2. Correction manuelle
                           | 3. Suppression
----------------------------|-----------------------------------
Metadata corrompue         | 1. Restore backup
                           | 2. --repair
                           | 3. Resync replica set
----------------------------|-----------------------------------
Corruption partielle       | 1. Export données valides
                           | 2. Rebuild database
                           | 3. Import données
```

### Procédure Standard de Récupération

```bash
#!/bin/bash
# Script de récupération standard

set -e  # Arrêter en cas d'erreur

BACKUP_DIR="/backup/mongodb"
DATA_DIR="/var/lib/mongodb"
CORRUPT_DIR="/var/lib/mongodb.corrupted.$(date +%Y%m%d_%H%M%S)"
LOG_FILE="/var/log/mongodb-recovery.log"

echo "=== MongoDB Recovery Procedure ===" | tee -a $LOG_FILE
echo "Start time: $(date)" | tee -a $LOG_FILE

# 1. Arrêter MongoDB
echo "Step 1: Stopping MongoDB..." | tee -a $LOG_FILE
sudo systemctl stop mongod

# 2. Sauvegarder l'état corrompu
echo "Step 2: Backing up corrupted state..." | tee -a $LOG_FILE
sudo mv $DATA_DIR $CORRUPT_DIR
sudo mkdir -p $DATA_DIR
sudo chown mongodb:mongodb $DATA_DIR

# 3. Déterminer le meilleur backup
echo "Step 3: Finding latest backup..." | tee -a $LOG_FILE
LATEST_BACKUP=$(ls -td $BACKUP_DIR/snapshot-* | head -1)
echo "Using backup: $LATEST_BACKUP" | tee -a $LOG_FILE

# 4. Restaurer le backup
echo "Step 4: Restoring backup..." | tee -a $LOG_FILE
sudo rsync -av --progress $LATEST_BACKUP/ $DATA_DIR/ 2>&1 | tee -a $LOG_FILE

# 5. Vérifier les permissions
echo "Step 5: Fixing permissions..." | tee -a $LOG_FILE
sudo chown -R mongodb:mongodb $DATA_DIR

# 6. Redémarrer MongoDB
echo "Step 6: Starting MongoDB..." | tee -a $LOG_FILE
sudo systemctl start mongod

# Attendre le démarrage
sleep 10

# 7. Vérifier le fonctionnement
echo "Step 7: Verifying MongoDB..." | tee -a $LOG_FILE
mongosh --eval "db.adminCommand({ping: 1})" 2>&1 | tee -a $LOG_FILE

# 8. Valider les données
echo "Step 8: Validating data..." | tee -a $LOG_FILE
mongosh --eval "
  var dbs = db.adminCommand({listDatabases: 1}).databases
  dbs.forEach(function(dbInfo) {
    if (!['admin', 'local', 'config'].includes(dbInfo.name)) {
      print('Validating database: ' + dbInfo.name)
      var db = db.getSiblingDB(dbInfo.name)
      db.getCollectionNames().forEach(function(collName) {
        print('  Validating collection: ' + collName)
        var result = db[collName].validate()
        if (!result.valid) {
          print('    ERROR: Invalid collection!')
        }
      })
    }
  })
" 2>&1 | tee -a $LOG_FILE

echo "End time: $(date)" | tee -a $LOG_FILE
echo "Recovery complete. Check $LOG_FILE for details."
echo "Corrupted data saved in: $CORRUPT_DIR"
```

### Checklist Post-Récupération

```javascript
// Vérifications après récupération

// 1. Vérifier le statut du serveur
db.serverStatus()

// 2. Valider toutes les collections
function validateAll() {
  var results = {valid: [], invalid: []}

  db.adminCommand({listDatabases: 1}).databases.forEach(dbInfo => {
    if (!['admin', 'local', 'config'].includes(dbInfo.name)) {
      var db = db.getSiblingDB(dbInfo.name)

      db.getCollectionNames().forEach(collName => {
        var validation = db[collName].validate({full: true})

        if (validation.valid) {
          results.valid.push(dbInfo.name + '.' + collName)
        } else {
          results.invalid.push({
            ns: dbInfo.name + '.' + collName,
            errors: validation.errors
          })
        }
      })
    }
  })

  return results
}

var validationResults = validateAll()
print("Valid collections:", validationResults.valid.length)
print("Invalid collections:", validationResults.invalid.length)

if (validationResults.invalid.length > 0) {
  print("\nInvalid collections:")
  printjson(validationResults.invalid)
}

// 3. Vérifier les counts
db.getCollectionNames().forEach(collName => {
  print(collName + ":", db[collName].countDocuments({}))
})

// 4. Vérifier les index
db.getCollectionNames().forEach(collName => {
  var indexes = db[collName].getIndexes()
  print(collName + ": " + indexes.length + " indexes")
})

// 5. Tester les opérations CRUD
try {
  var testDoc = {test: true, timestamp: new Date()}
  db.test_recovery.insertOne(testDoc)
  db.test_recovery.findOne({test: true})
  db.test_recovery.updateOne({test: true}, {$set: {updated: true}})
  db.test_recovery.deleteOne({test: true})
  db.test_recovery.drop()
  print("✓ CRUD operations working")
} catch (e) {
  print("✗ CRUD operations failed:", e.message)
}

// 6. Pour replica set, vérifier la réplication
try {
  rs.status()
  rs.printReplicationInfo()
  print("✓ Replication status OK")
} catch (e) {
  print("Not a replica set or replication issue:", e.message)
}
```

---

## 8. Prévention

### Mesures Préventives

#### 1. Backups Réguliers et Testés

```bash
#!/bin/bash
# Script de backup automatisé

BACKUP_DIR="/backup/mongodb"
DATE=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_DIR="$BACKUP_DIR/snapshot-$DATE"

# Créer le répertoire
mkdir -p $SNAPSHOT_DIR

# Snapshot filesystem (méthode préférée)
# Freeze filesystem
mongosh --eval "db.fsyncLock()"

# Créer snapshot (exemple LVM)
lvcreate -L 10G -s -n mongodb_snap /dev/vg0/mongodb_lv

# Unfreeze
mongosh --eval "db.fsyncUnlock()"

# Monter et copier le snapshot
mount /dev/vg0/mongodb_snap /mnt/snapshot
rsync -av /mnt/snapshot/ $SNAPSHOT_DIR/
umount /mnt/snapshot
lvremove -f /dev/vg0/mongodb_snap

# Ou utiliser mongodump (plus lent mais universel)
mongodump --out $SNAPSHOT_DIR/dump/ --oplog

# Nettoyer les anciens backups (garder 7 jours)
find $BACKUP_DIR -type d -name "snapshot-*" -mtime +7 -exec rm -rf {} \;

# Tester le backup (IMPORTANT!)
echo "Testing backup..."
mongorestore --dryRun $SNAPSHOT_DIR/dump/ > /tmp/restore-test.log 2>&1

if [ $? -eq 0 ]; then
  echo "Backup verified successfully"
else
  echo "Backup verification FAILED!" | mail -s "Backup Alert" admin@example.com
fi
```

#### 2. Monitoring Proactif

```javascript
// Script de monitoring de santé

function healthCheck() {
  var health = {
    timestamp: new Date(),
    alerts: []
  }

  // 1. Vérifier les erreurs dans les logs
  var logErrors = db.adminCommand({getLog: "global"}).log.filter(line =>
    line.includes("ERROR") || line.includes("FATAL") || line.includes("corrupt")
  ).length

  if (logErrors > 0) {
    health.alerts.push(`${logErrors} errors found in logs`)
  }

  // 2. Vérifier l'espace disque
  var fsStats = db.serverStatus().extra_info
  // Implémenter vérification espace disque

  // 3. Vérifier la réplication (si applicable)
  try {
    var replStatus = rs.status()
    replStatus.members.forEach(member => {
      if (member.health !== 1) {
        health.alerts.push(`${member.name} is unhealthy`)
      }
    })
  } catch (e) {
    // Pas un replica set
  }

  // 4. Vérifier les métriques WiredTiger
  var wtStats = db.serverStatus().wiredTiger
  if (wtStats.log && wtStats.log["log sync operations"] > 1000000) {
    health.alerts.push("High log sync operations - potential I/O issues")
  }

  // 5. Envoyer alertes
  if (health.alerts.length > 0) {
    // Envoyer à système de monitoring
    print("ALERTS:")
    health.alerts.forEach(alert => print("  - " + alert))
  }

  return health
}

// Exécuter périodiquement
setInterval(healthCheck, 60000)  // Chaque minute
```

#### 3. Configuration Robuste

```yaml
# /etc/mongod.conf

# Journaling activé (CRITIQUE pour durabilité)
storage:
  journal:
    enabled: true
    commitIntervalMs: 100

  # Checksums activés
  wiredTiger:
    engineConfig:
      directoryForIndexes: true

    collectionConfig:
      blockCompressor: snappy

    indexConfig:
      prefixCompression: true

# Logging détaillé
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  verbosity: 1  # Augmenter temporairement si problèmes

# Write Concern approprié
replication:
  replSetName: rs0

# Pour production TOUJOURS utiliser replica set
# même avec 1 seul nœud + 2 arbiters pour durabilité
```

#### 4. Infrastructure Fiable

```bash
# Matériel recommandé

# 1. Disques
# - RAID 10 pour données MongoDB
# - SSD pour meilleure performance et fiabilité
# - Éviter RAID 5/6 (pénalité écriture)

# 2. Vérifier régulièrement les disques
sudo smartctl -H /dev/sda
sudo smartctl -a /dev/sda | grep -i "reallocated\|pending\|uncorrectable"

# 3. Activer les checksums filesystem
# ext4 : metadata_csum
# xfs : par défaut

# 4. Monitoring hardware
# Configurer des alertes sur :
# - Température CPU/Disques
# - Erreurs ECC mémoire
# - Erreurs SMART
# - Dégradation RAID
```

#### 5. Tests Réguliers de Récupération

```bash
#!/bin/bash
# Test mensuel de procédure de récupération

# 1. Prendre un backup de test
mongodump --out /tmp/test-backup/

# 2. Restaurer sur environnement de test
mongorestore --host test-mongodb:27017 --drop /tmp/test-backup/

# 3. Valider
mongosh test-mongodb:27017 --eval "
  db.getCollectionNames().forEach(coll => {
    var result = db[coll].validate({full: true})
    if (!result.valid) {
      print('FAIL: ' + coll)
      quit(1)
    }
  })
  print('SUCCESS: All collections valid')
"

# 4. Mesurer le temps
# RPO (Recovery Point Objective)
# RTO (Recovery Time Objective)

# 5. Documenter
echo "Recovery test completed: $(date)" >> /var/log/recovery-tests.log
```

---

## Checklist d'Urgence Corruption

```
INCIDENT: Corruption de données détectée

☐ IMMÉDIAT (5 minutes)
  ☐ Isoler le serveur affecté (arrêter trafic si possible)
  ☐ Capturer les logs actuels
  ☐ Vérifier si replica set (utiliser secondary)
  ☐ Alerter l'équipe

☐ DIAGNOSTIC (15 minutes)
  ☐ Identifier le type de corruption (fichier/index/doc/metadata)
  ☐ Évaluer l'étendue (validate collections)
  ☐ Vérifier le hardware (disques, mémoire)
  ☐ Localiser le dernier backup valide

☐ DÉCISION (5 minutes)
  ☐ Restore depuis backup ? (si dispo et récent)
  ☐ Resync depuis replica set ? (si membre d'un RS)
  ☐ Réparation en place ? (dernier recours)
  ☐ Récupération partielle ?

☐ EXÉCUTION (30-120 minutes)
  ☐ Suivre la procédure choisie
  ☐ Documenter chaque étape
  ☐ Vérifier la progression
  ☐ Tester les données restaurées

☐ VALIDATION (30 minutes)
  ☐ validate() toutes les collections
  ☐ Vérifier les counts
  ☐ Tester CRUD operations
  ☐ Vérifier réplication (si RS)

☐ POST-INCIDENT
  ☐ Analyse de cause racine
  ☐ Rapport d'incident
  ☐ Améliorer les procédures
  ☐ Corriger les problèmes hardware/software
```

---

## Conclusion

La corruption de données est un incident grave qui nécessite :

1. **Préparation** : Backups testés, replica sets, monitoring
2. **Détection rapide** : Validation régulière, alertes
3. **Procédures claires** : Documentation, scripts, tests
4. **Infrastructure robuste** : Hardware fiable, configuration appropriée

**Principes clés :**
- ✅ **Backups testés régulièrement** (RPO/RTO connus)
- ✅ **Replica Set TOUJOURS** (minimum 3 nœuds)
- ✅ **Journaling activé** (durabilité des écritures)
- ✅ **Monitoring proactif** (détecter avant corruption)
- ✅ **Hardware fiable** (SSD, RAID 10, ECC RAM)
- ✅ **Validation régulière** (scheduled validates)
- ✅ **Procédures testées** (DR drills mensuels)

**L'objectif est de PRÉVENIR plutôt que GUÉRIR.**

---


⏭️ [Récupération après panne](/22-depannage-resolution-problemes/06-recuperation-apres-panne.md)
