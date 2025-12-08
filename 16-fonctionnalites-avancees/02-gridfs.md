🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.2 GridFS (Stockage de fichiers volumineux)

## Introduction

**GridFS** est une spécification MongoDB pour stocker et récupérer des fichiers qui dépassent la limite BSON de 16 mégaoctets. Plutôt qu'une collection MongoDB traditionnelle, GridFS divise les fichiers en chunks (morceaux) et les stocke dans deux collections distinctes : `fs.files` pour les métadonnées et `fs.chunks` pour les données binaires.

GridFS est particulièrement utile lorsque vous souhaitez stocker des fichiers volumineux tout en bénéficiant des avantages de MongoDB : réplication automatique, sharding, requêtes sur métadonnées, et intégration native avec votre base de données.

---

## Architecture et fonctionnement

### Structure de GridFS

```
┌──────────────────────────────────────────────────────────┐
│                        GridFS                            │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │           Collection: fs.files                   │    │
│  │  (Métadonnées des fichiers)                      │    │
│  │                                                  │    │
│  │  {                                               │    │
│  │    _id: ObjectId("..."),                         │    │
│  │    length: 5242880,      // Taille totale        │    │
│  │    chunkSize: 261120,    // Taille d'un chunk    │    │
│  │    uploadDate: ISODate,                          │    │
│  │    filename: "video.mp4",                        │    │
│  │    contentType: "video/mp4",                     │    │
│  │    metadata: { /* custom */ }                    │    │
│  │  }                                               │    │
│  └──────────────────────────────────────────────────┘    │
│                          │                               │
│                          │ _id reference                 │
│                          ▼                               │
│  ┌──────────────────────────────────────────────────┐    │
│  │           Collection: fs.chunks                  │    │
│  │  (Données binaires fragmentées)                  │    │
│  │                                                  │    │
│  │  {                                               │    │
│  │    _id: ObjectId("..."),                         │    │
│  │    files_id: ObjectId("..."), // Ref to fs.files │    │
│  │    n: 0,                      // Numéro chunk    │    │
│  │    data: BinData(...)         // Données         │    │
│  │  }                                               │    │
│  │  { files_id: ..., n: 1, data: ... }              │    │
│  │  { files_id: ..., n: 2, data: ... }              │    │
│  │  ...                                             │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Taille des chunks

- **Par défaut** : 255 Ko (261120 bytes)
- **Configurable** : Peut être ajusté selon les besoins
- **Compromis** :
  - Chunks plus petits : Plus de documents, overhead plus important
  - Chunks plus grands : Moins de documents, transferts moins granulaires

### Index automatiques

GridFS crée automatiquement des index pour optimiser les performances :

```javascript
// Sur fs.files
{ _id: 1 }
{ filename: 1, uploadDate: 1 }

// Sur fs.chunks
{ files_id: 1, n: 1 }  // Composé et unique
```

---

## API GridFS et opérations de base

### Initialisation

```javascript
const { MongoClient, GridFSBucket } = require('mongodb');

const client = await MongoClient.connect('mongodb://localhost:27017');
const db = client.db('myapp');

// Créer un bucket GridFS (collection fs.files et fs.chunks)
const bucket = new GridFSBucket(db, {
  bucketName: 'fs',        // Préfixe des collections (défaut: 'fs')
  chunkSizeBytes: 261120   // Taille des chunks (défaut: 255KB)
});
```

### Upload de fichiers

```javascript
const fs = require('fs');
const path = require('path');

// Méthode 1 : Upload depuis un stream
async function uploadFromStream(filePath) {
  const filename = path.basename(filePath);

  // Créer un stream de lecture
  const readStream = fs.createReadStream(filePath);

  // Créer un stream d'upload GridFS
  const uploadStream = bucket.openUploadStream(filename, {
    contentType: 'video/mp4',
    metadata: {
      uploadedBy: 'user-123',
      category: 'media',
      tags: ['important', 'demo'],
      originalPath: filePath
    }
  });

  // Pipe et attendre la fin
  return new Promise((resolve, reject) => {
    readStream.pipe(uploadStream)
      .on('error', reject)
      .on('finish', () => {
        console.log('File uploaded:', uploadStream.id);
        resolve(uploadStream.id);
      });
  });
}

// Utilisation
const fileId = await uploadFromStream('./video.mp4');
console.log('File ID:', fileId);


// Méthode 2 : Upload depuis un buffer
async function uploadFromBuffer(buffer, filename, options = {}) {
  const uploadStream = bucket.openUploadStream(filename, {
    contentType: options.contentType || 'application/octet-stream',
    metadata: options.metadata || {}
  });

  return new Promise((resolve, reject) => {
    uploadStream.end(buffer, (error) => {
      if (error) {
        reject(error);
      } else {
        resolve(uploadStream.id);
      }
    });
  });
}

// Utilisation
const imageBuffer = await fs.promises.readFile('./image.jpg');
const imageId = await uploadFromBuffer(imageBuffer, 'profile.jpg', {
  contentType: 'image/jpeg',
  metadata: { userId: 'user-456' }
});
```

### Download de fichiers

```javascript
// Méthode 1 : Download vers un stream
async function downloadToStream(fileId, outputPath) {
  const downloadStream = bucket.openDownloadStream(fileId);
  const writeStream = fs.createWriteStream(outputPath);

  return new Promise((resolve, reject) => {
    downloadStream.pipe(writeStream)
      .on('error', reject)
      .on('finish', resolve);
  });
}

// Utilisation
await downloadToStream(fileId, './downloaded-video.mp4');


// Méthode 2 : Download vers un buffer
async function downloadToBuffer(fileId) {
  const downloadStream = bucket.openDownloadStream(fileId);
  const chunks = [];

  return new Promise((resolve, reject) => {
    downloadStream
      .on('data', (chunk) => chunks.push(chunk))
      .on('error', reject)
      .on('end', () => resolve(Buffer.concat(chunks)));
  });
}

// Utilisation
const fileBuffer = await downloadToBuffer(fileId);


// Méthode 3 : Download par nom de fichier
async function downloadByName(filename, outputPath) {
  const downloadStream = bucket.openDownloadStreamByName(filename, {
    revision: -1  // -1 = plus récent, 0 = plus ancien, n = n-ième version
  });

  const writeStream = fs.createWriteStream(outputPath);

  return new Promise((resolve, reject) => {
    downloadStream.pipe(writeStream)
      .on('error', reject)
      .on('finish', resolve);
  });
}
```

### Suppression de fichiers

```javascript
// Supprimer par ID (supprime le document fs.files ET tous les chunks)
async function deleteFile(fileId) {
  try {
    await bucket.delete(fileId);
    console.log('File deleted:', fileId);
  } catch (error) {
    if (error.code === 'ENOENT') {
      console.log('File not found');
    } else {
      throw error;
    }
  }
}

// Suppression multiple (par requête)
async function deleteFilesByQuery(query) {
  const files = await db.collection('fs.files')
    .find(query)
    .project({ _id: 1 })
    .toArray();

  for (const file of files) {
    await bucket.delete(file._id);
  }

  console.log(`Deleted ${files.length} files`);
}

// Utilisation
await deleteFilesByQuery({
  'metadata.userId': 'user-123',
  uploadDate: { $lt: new Date('2024-01-01') }
});
```

### Requêtes sur métadonnées

```javascript
// Rechercher des fichiers par métadonnées
async function findFiles(query, options = {}) {
  return await db.collection('fs.files')
    .find(query)
    .sort(options.sort || { uploadDate: -1 })
    .limit(options.limit || 20)
    .toArray();
}

// Exemples de requêtes
const userFiles = await findFiles({ 'metadata.userId': 'user-123' });

const recentVideos = await findFiles({
  contentType: 'video/mp4',
  uploadDate: { $gte: new Date('2024-12-01') }
});

const largeFiles = await findFiles({
  length: { $gt: 100 * 1024 * 1024 } // > 100 MB
});

// Requête complexe avec agrégation
const fileStats = await db.collection('fs.files').aggregate([
  {
    $match: { 'metadata.category': 'media' }
  },
  {
    $group: {
      _id: '$contentType',
      count: { $sum: 1 },
      totalSize: { $sum: '$length' },
      avgSize: { $avg: '$length' }
    }
  }
]).toArray();
```

---

## Cas d'usage avancés

### Cas 1 : Système de gestion de médias avec streaming

```javascript
class MediaStorageService {
  constructor(db) {
    this.db = db;
    this.bucket = new GridFSBucket(db, {
      bucketName: 'media',
      chunkSizeBytes: 512 * 1024 // 512 KB pour streaming vidéo
    });
  }

  async uploadMedia(filePath, metadata) {
    const stats = await fs.promises.stat(filePath);
    const filename = path.basename(filePath);

    // Détection du type MIME
    const contentType = this.detectContentType(filename);

    // Upload avec métadonnées enrichies
    const uploadStream = this.bucket.openUploadStream(filename, {
      contentType,
      metadata: {
        ...metadata,
        originalName: filename,
        fileSize: stats.size,
        uploadedAt: new Date(),
        processingStatus: 'pending'
      }
    });

    const readStream = fs.createReadStream(filePath);

    return new Promise((resolve, reject) => {
      readStream.pipe(uploadStream)
        .on('error', reject)
        .on('finish', async () => {
          // Post-processing asynchrone
          await this.scheduleProcessing(uploadStream.id, contentType);
          resolve({
            fileId: uploadStream.id,
            filename,
            contentType,
            size: stats.size
          });
        });
    });
  }

  async streamToClient(fileId, response, options = {}) {
    try {
      // Récupérer les métadonnées
      const file = await this.db.collection('media.files')
        .findOne({ _id: fileId });

      if (!file) {
        response.status(404).send('File not found');
        return;
      }

      // Support du Range header pour streaming vidéo
      const range = options.range;
      let downloadStream;

      if (range) {
        // Parse du range header
        const parts = range.replace(/bytes=/, '').split('-');
        const start = parseInt(parts[0], 10);
        const end = parts[1] ? parseInt(parts[1], 10) : file.length - 1;
        const chunksize = (end - start) + 1;

        // Headers pour partial content
        response.status(206);
        response.set({
          'Content-Range': `bytes ${start}-${end}/${file.length}`,
          'Accept-Ranges': 'bytes',
          'Content-Length': chunksize,
          'Content-Type': file.contentType
        });

        // Stream avec range
        downloadStream = this.bucket.openDownloadStream(fileId, {
          start,
          end: end + 1
        });
      } else {
        // Stream complet
        response.set({
          'Content-Type': file.contentType,
          'Content-Length': file.length,
          'Content-Disposition': `inline; filename="${file.filename}"`,
          'Cache-Control': 'public, max-age=31536000' // 1 an
        });

        downloadStream = this.bucket.openDownloadStream(fileId);
      }

      // Pipe vers la réponse
      downloadStream.pipe(response);

      downloadStream.on('error', (error) => {
        console.error('Stream error:', error);
        if (!response.headersSent) {
          response.status(500).send('Stream error');
        }
      });

    } catch (error) {
      console.error('Streaming error:', error);
      response.status(500).send('Internal server error');
    }
  }

  async scheduleProcessing(fileId, contentType) {
    // Marquer pour processing
    await this.db.collection('media.files').updateOne(
      { _id: fileId },
      {
        $set: {
          'metadata.processingStatus': 'queued',
          'metadata.queuedAt': new Date()
        }
      }
    );

    // Ajouter à une queue de traitement (ex: pour génération de thumbnails)
    if (contentType.startsWith('image/')) {
      await this.queueThumbnailGeneration(fileId);
    } else if (contentType.startsWith('video/')) {
      await this.queueVideoTranscoding(fileId);
    }
  }

  async queueThumbnailGeneration(fileId) {
    // Intégration avec système de queue (Bull, BullMQ, etc.)
    console.log('Queued thumbnail generation for:', fileId);
  }

  async queueVideoTranscoding(fileId) {
    console.log('Queued video transcoding for:', fileId);
  }

  detectContentType(filename) {
    const ext = path.extname(filename).toLowerCase();
    const mimeTypes = {
      '.jpg': 'image/jpeg',
      '.jpeg': 'image/jpeg',
      '.png': 'image/png',
      '.gif': 'image/gif',
      '.mp4': 'video/mp4',
      '.webm': 'video/webm',
      '.pdf': 'application/pdf',
      '.zip': 'application/zip'
    };
    return mimeTypes[ext] || 'application/octet-stream';
  }

  async getMediaInfo(fileId) {
    const file = await this.db.collection('media.files')
      .findOne({ _id: fileId });

    if (!file) {
      return null;
    }

    return {
      id: file._id,
      filename: file.filename,
      contentType: file.contentType,
      size: file.length,
      uploadDate: file.uploadDate,
      metadata: file.metadata
    };
  }

  async deleteMedia(fileId) {
    // Supprimer les fichiers dérivés (thumbnails, transcodes, etc.)
    await this.deleteDerivatives(fileId);

    // Supprimer le fichier principal
    await this.bucket.delete(fileId);
  }

  async deleteDerivatives(fileId) {
    // Supprimer thumbnails, versions transcodées, etc.
    const derivatives = await this.db.collection('media.files')
      .find({ 'metadata.originalFileId': fileId })
      .toArray();

    for (const derivative of derivatives) {
      await this.bucket.delete(derivative._id);
    }
  }
}

// Utilisation avec Express
const express = require('express');
const app = express();

const mediaService = new MediaStorageService(db);

// Upload endpoint
app.post('/media/upload', upload.single('file'), async (req, res) => {
  try {
    const result = await mediaService.uploadMedia(req.file.path, {
      userId: req.user.id,
      category: req.body.category,
      tags: req.body.tags
    });

    res.json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Streaming endpoint
app.get('/media/:fileId', async (req, res) => {
  await mediaService.streamToClient(
    new ObjectId(req.params.fileId),
    res,
    { range: req.headers.range }
  );
});

// Info endpoint
app.get('/media/:fileId/info', async (req, res) => {
  const info = await mediaService.getMediaInfo(
    new ObjectId(req.params.fileId)
  );

  if (!info) {
    return res.status(404).json({ error: 'File not found' });
  }

  res.json(info);
});
```

### Cas 2 : Versioning de fichiers

```javascript
class VersionedFileStorage {
  constructor(db) {
    this.db = db;
    this.bucket = new GridFSBucket(db, { bucketName: 'documents' });
  }

  async uploadVersion(filePath, documentId, version, metadata = {}) {
    const filename = path.basename(filePath);

    // Nom versionné
    const versionedFilename = `${documentId}_v${version}_${filename}`;

    const uploadStream = this.bucket.openUploadStream(versionedFilename, {
      contentType: this.detectContentType(filename),
      metadata: {
        documentId,
        version,
        originalFilename: filename,
        uploadedAt: new Date(),
        uploadedBy: metadata.userId,
        changeDescription: metadata.changeDescription || '',
        ...metadata
      }
    });

    const readStream = fs.createReadStream(filePath);

    return new Promise((resolve, reject) => {
      readStream.pipe(uploadStream)
        .on('error', reject)
        .on('finish', async () => {
          // Enregistrer dans l'historique
          await this.recordVersion({
            documentId,
            version,
            fileId: uploadStream.id,
            filename: versionedFilename,
            uploadedBy: metadata.userId,
            changeDescription: metadata.changeDescription
          });

          resolve({
            fileId: uploadStream.id,
            documentId,
            version
          });
        });
    });
  }

  async recordVersion(versionInfo) {
    await this.db.collection('document_versions').insertOne({
      ...versionInfo,
      createdAt: new Date()
    });
  }

  async getVersionHistory(documentId) {
    return await this.db.collection('document_versions')
      .find({ documentId })
      .sort({ version: -1 })
      .toArray();
  }

  async downloadVersion(documentId, version, outputPath) {
    const versionInfo = await this.db.collection('document_versions')
      .findOne({ documentId, version });

    if (!versionInfo) {
      throw new Error('Version not found');
    }

    const downloadStream = this.bucket.openDownloadStream(versionInfo.fileId);
    const writeStream = fs.createWriteStream(outputPath);

    return new Promise((resolve, reject) => {
      downloadStream.pipe(writeStream)
        .on('error', reject)
        .on('finish', resolve);
    });
  }

  async getLatestVersion(documentId) {
    const versions = await this.getVersionHistory(documentId);
    return versions[0]; // Premier = plus récent
  }

  async compareVersions(documentId, version1, version2) {
    const [v1, v2] = await Promise.all([
      this.db.collection('document_versions')
        .findOne({ documentId, version: version1 }),
      this.db.collection('document_versions')
        .findOne({ documentId, version: version2 })
    ]);

    if (!v1 || !v2) {
      throw new Error('Version not found');
    }

    return {
      version1: {
        version: v1.version,
        uploadedAt: v1.createdAt,
        uploadedBy: v1.uploadedBy,
        changeDescription: v1.changeDescription
      },
      version2: {
        version: v2.version,
        uploadedAt: v2.createdAt,
        uploadedBy: v2.uploadedBy,
        changeDescription: v2.changeDescription
      }
    };
  }

  async revertToVersion(documentId, targetVersion, userId) {
    const versionInfo = await this.db.collection('document_versions')
      .findOne({ documentId, version: targetVersion });

    if (!versionInfo) {
      throw new Error('Target version not found');
    }

    // Récupérer le fichier de la version cible
    const buffer = await this.downloadToBuffer(versionInfo.fileId);

    // Créer une nouvelle version (pas d'écrasement)
    const latestVersion = await this.getLatestVersion(documentId);
    const newVersion = (latestVersion?.version || 0) + 1;

    // Upload comme nouvelle version
    const uploadStream = this.bucket.openUploadStream(
      `${documentId}_v${newVersion}_${versionInfo.originalFilename}`,
      {
        contentType: versionInfo.contentType,
        metadata: {
          documentId,
          version: newVersion,
          originalFilename: versionInfo.originalFilename,
          uploadedBy: userId,
          changeDescription: `Reverted to version ${targetVersion}`,
          revertedFrom: targetVersion
        }
      }
    );

    return new Promise((resolve, reject) => {
      uploadStream.end(buffer, async (error) => {
        if (error) {
          reject(error);
        } else {
          await this.recordVersion({
            documentId,
            version: newVersion,
            fileId: uploadStream.id,
            filename: `${documentId}_v${newVersion}_${versionInfo.originalFilename}`,
            uploadedBy: userId,
            changeDescription: `Reverted to version ${targetVersion}`
          });

          resolve({
            fileId: uploadStream.id,
            version: newVersion,
            revertedFrom: targetVersion
          });
        }
      });
    });
  }

  async downloadToBuffer(fileId) {
    const downloadStream = this.bucket.openDownloadStream(fileId);
    const chunks = [];

    return new Promise((resolve, reject) => {
      downloadStream
        .on('data', (chunk) => chunks.push(chunk))
        .on('error', reject)
        .on('end', () => resolve(Buffer.concat(chunks)));
    });
  }

  async deleteOldVersions(documentId, keepLast = 5) {
    const versions = await this.getVersionHistory(documentId);

    // Garder seulement les N dernières versions
    const toDelete = versions.slice(keepLast);

    for (const version of toDelete) {
      await this.bucket.delete(version.fileId);
      await this.db.collection('document_versions').deleteOne({
        _id: version._id
      });
    }

    console.log(`Deleted ${toDelete.length} old versions of ${documentId}`);
  }

  detectContentType(filename) {
    // Même logique que précédemment
    const ext = path.extname(filename).toLowerCase();
    const mimeTypes = {
      '.pdf': 'application/pdf',
      '.docx': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
      '.xlsx': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    };
    return mimeTypes[ext] || 'application/octet-stream';
  }
}

// Utilisation
const versionedStorage = new VersionedFileStorage(db);

// Upload version 1
await versionedStorage.uploadVersion(
  './contract.pdf',
  'DOC-123',
  1,
  { userId: 'user-456', changeDescription: 'Initial version' }
);

// Upload version 2
await versionedStorage.uploadVersion(
  './contract-v2.pdf',
  'DOC-123',
  2,
  { userId: 'user-456', changeDescription: 'Added signature page' }
);

// Historique
const history = await versionedStorage.getVersionHistory('DOC-123');

// Revert
await versionedStorage.revertToVersion('DOC-123', 1, 'user-456');

// Nettoyage
await versionedStorage.deleteOldVersions('DOC-123', 5);
```

### Cas 3 : CDN-like avec cache et compression

```javascript
const zlib = require('zlib');
const { createHash } = require('crypto');

class GridFSCDN {
  constructor(db, redisClient) {
    this.db = db;
    this.redis = redisClient;
    this.bucket = new GridFSBucket(db, { bucketName: 'cdn' });
  }

  async uploadWithCompression(filePath, options = {}) {
    const filename = path.basename(filePath);
    const stats = await fs.promises.stat(filePath);

    // Lire le fichier
    const buffer = await fs.promises.readFile(filePath);

    // Calculer le hash pour déduplication
    const hash = createHash('sha256').update(buffer).digest('hex');

    // Vérifier si déjà uploadé (déduplication)
    const existing = await this.db.collection('cdn.files')
      .findOne({ 'metadata.sha256': hash });

    if (existing && options.deduplicate !== false) {
      console.log('File already exists, reusing:', existing._id);
      return {
        fileId: existing._id,
        deduplicated: true,
        existingFileId: existing._id
      };
    }

    // Compresser si fichier texte/JSON/HTML
    let finalBuffer = buffer;
    let compressed = false;
    const compressibleTypes = ['text/', 'application/json', 'application/javascript'];

    if (compressibleTypes.some(type => options.contentType?.startsWith(type))) {
      finalBuffer = await this.compressBuffer(buffer);
      compressed = true;
      console.log(`Compressed: ${buffer.length} -> ${finalBuffer.length} bytes`);
    }

    // Upload
    const uploadStream = this.bucket.openUploadStream(filename, {
      contentType: options.contentType || 'application/octet-stream',
      metadata: {
        originalSize: stats.size,
        compressedSize: finalBuffer.length,
        compressed,
        sha256: hash,
        uploadedAt: new Date(),
        ...options.metadata
      }
    });

    return new Promise((resolve, reject) => {
      uploadStream.end(finalBuffer, (error) => {
        if (error) {
          reject(error);
        } else {
          resolve({
            fileId: uploadStream.id,
            compressed,
            originalSize: stats.size,
            storedSize: finalBuffer.length,
            compressionRatio: compressed
              ? ((1 - finalBuffer.length / stats.size) * 100).toFixed(2) + '%'
              : '0%'
          });
        }
      });
    });
  }

  async serveWithCache(fileId, response) {
    try {
      // 1. Vérifier le cache Redis
      const cacheKey = `cdn:${fileId}`;
      const cached = await this.redis.get(cacheKey);

      if (cached) {
        console.log('Cache HIT:', fileId);
        const data = JSON.parse(cached);

        response.set({
          'Content-Type': data.contentType,
          'Content-Length': data.content.length,
          'Cache-Control': 'public, max-age=31536000',
          'X-Cache': 'HIT'
        });

        return response.send(Buffer.from(data.content, 'base64'));
      }

      console.log('Cache MISS:', fileId);

      // 2. Récupérer depuis GridFS
      const file = await this.db.collection('cdn.files')
        .findOne({ _id: fileId });

      if (!file) {
        return response.status(404).send('File not found');
      }

      // 3. Download
      const buffer = await this.downloadToBuffer(fileId);

      // 4. Décompresser si nécessaire
      let finalBuffer = buffer;
      if (file.metadata.compressed) {
        finalBuffer = await this.decompressBuffer(buffer);
      }

      // 5. Mettre en cache
      await this.redis.setex(
        cacheKey,
        86400, // 24h
        JSON.stringify({
          contentType: file.contentType,
          content: finalBuffer.toString('base64')
        })
      );

      // 6. Répondre
      response.set({
        'Content-Type': file.contentType,
        'Content-Length': finalBuffer.length,
        'Cache-Control': 'public, max-age=31536000',
        'X-Cache': 'MISS',
        'X-Original-Size': file.metadata.originalSize,
        'X-Compressed': file.metadata.compressed
      });

      response.send(finalBuffer);

    } catch (error) {
      console.error('CDN serve error:', error);
      response.status(500).send('Internal server error');
    }
  }

  async compressBuffer(buffer) {
    return new Promise((resolve, reject) => {
      zlib.gzip(buffer, { level: 9 }, (error, result) => {
        if (error) reject(error);
        else resolve(result);
      });
    });
  }

  async decompressBuffer(buffer) {
    return new Promise((resolve, reject) => {
      zlib.gunzip(buffer, (error, result) => {
        if (error) reject(error);
        else resolve(result);
      });
    });
  }

  async downloadToBuffer(fileId) {
    const downloadStream = this.bucket.openDownloadStream(fileId);
    const chunks = [];

    return new Promise((resolve, reject) => {
      downloadStream
        .on('data', (chunk) => chunks.push(chunk))
        .on('error', reject)
        .on('end', () => resolve(Buffer.concat(chunks)));
    });
  }

  async invalidateCache(fileId) {
    const cacheKey = `cdn:${fileId}`;
    await this.redis.del(cacheKey);
    console.log('Cache invalidated:', fileId);
  }

  async getStatistics() {
    const stats = await this.db.collection('cdn.files').aggregate([
      {
        $group: {
          _id: null,
          totalFiles: { $sum: 1 },
          totalOriginalSize: { $sum: '$metadata.originalSize' },
          totalStoredSize: { $sum: '$metadata.compressedSize' },
          compressedFiles: {
            $sum: { $cond: ['$metadata.compressed', 1, 0] }
          }
        }
      }
    ]).toArray();

    if (stats.length === 0) {
      return null;
    }

    const s = stats[0];
    return {
      totalFiles: s.totalFiles,
      totalOriginalSize: s.totalOriginalSize,
      totalStoredSize: s.totalStoredSize,
      compressionRatio: ((1 - s.totalStoredSize / s.totalOriginalSize) * 100).toFixed(2) + '%',
      compressedFiles: s.compressedFiles,
      savedSpace: s.totalOriginalSize - s.totalStoredSize
    };
  }
}

// Utilisation
const cdn = new GridFSCDN(db, redisClient);

// Upload avec compression
const result = await cdn.uploadWithCompression(
  './app.bundle.js',
  {
    contentType: 'application/javascript',
    metadata: { version: '1.2.3' }
  }
);

console.log('Upload result:', result);

// Endpoint Express
app.get('/cdn/:fileId', async (req, res) => {
  await cdn.serveWithCache(new ObjectId(req.params.fileId), res);
});

// Statistiques
const stats = await cdn.getStatistics();
console.log('CDN Statistics:', stats);
```

### Cas 4 : Backup incrémental avec change streams

```javascript
class GridFSBackupService {
  constructor(sourceDb, backupDb) {
    this.sourceBucket = new GridFSBucket(sourceDb, { bucketName: 'production' });
    this.backupBucket = new GridFSBucket(backupDb, { bucketName: 'backup' });
    this.sourceDb = sourceDb;
    this.backupDb = backupDb;
  }

  async initializeIncrementalBackup() {
    // Surveiller les changements sur fs.files
    const changeStream = this.sourceDb
      .collection('production.files')
      .watch([
        {
          $match: {
            operationType: { $in: ['insert', 'update', 'delete'] }
          }
        }
      ], {
        fullDocument: 'updateLookup'
      });

    changeStream.on('change', async (change) => {
      await this.handleFileChange(change);
    });

    changeStream.on('error', (error) => {
      console.error('Backup stream error:', error);
      // Réinitialiser le stream
      setTimeout(() => this.initializeIncrementalBackup(), 5000);
    });

    console.log('Incremental backup initialized');
  }

  async handleFileChange(change) {
    try {
      switch (change.operationType) {
        case 'insert':
          await this.backupFile(change.fullDocument);
          break;

        case 'update':
          await this.updateBackup(change.fullDocument);
          break;

        case 'delete':
          await this.deleteBackup(change.documentKey._id);
          break;
      }

      // Logger le backup
      await this.logBackupOperation(change);
    } catch (error) {
      console.error('Backup operation failed:', error);
      await this.logBackupError(change, error);
    }
  }

  async backupFile(fileDoc) {
    console.log('Backing up file:', fileDoc._id);

    // Vérifier si déjà sauvegardé
    const existing = await this.backupDb
      .collection('backup.files')
      .findOne({ 'metadata.sourceId': fileDoc._id.toString() });

    if (existing) {
      console.log('File already backed up');
      return;
    }

    // Download depuis source
    const buffer = await this.downloadToBuffer(
      this.sourceBucket,
      fileDoc._id
    );

    // Upload vers backup
    const uploadStream = this.backupBucket.openUploadStream(
      fileDoc.filename,
      {
        contentType: fileDoc.contentType,
        metadata: {
          sourceId: fileDoc._id.toString(),
          backedUpAt: new Date(),
          originalMetadata: fileDoc.metadata
        }
      }
    );

    return new Promise((resolve, reject) => {
      uploadStream.end(buffer, (error) => {
        if (error) {
          reject(error);
        } else {
          console.log('Backup complete:', fileDoc._id);
          resolve(uploadStream.id);
        }
      });
    });
  }

  async updateBackup(fileDoc) {
    // Supprimer l'ancienne version
    const existing = await this.backupDb
      .collection('backup.files')
      .findOne({ 'metadata.sourceId': fileDoc._id.toString() });

    if (existing) {
      await this.backupBucket.delete(existing._id);
    }

    // Re-backup
    await this.backupFile(fileDoc);
  }

  async deleteBackup(fileId) {
    const backup = await this.backupDb
      .collection('backup.files')
      .findOne({ 'metadata.sourceId': fileId.toString() });

    if (backup) {
      await this.backupBucket.delete(backup._id);
      console.log('Backup deleted:', fileId);
    }
  }

  async downloadToBuffer(bucket, fileId) {
    const downloadStream = bucket.openDownloadStream(fileId);
    const chunks = [];

    return new Promise((resolve, reject) => {
      downloadStream
        .on('data', (chunk) => chunks.push(chunk))
        .on('error', reject)
        .on('end', () => resolve(Buffer.concat(chunks)));
    });
  }

  async logBackupOperation(change) {
    await this.backupDb.collection('backup_log').insertOne({
      operationType: change.operationType,
      fileId: change.documentKey._id,
      timestamp: new Date(),
      status: 'success'
    });
  }

  async logBackupError(change, error) {
    await this.backupDb.collection('backup_log').insertOne({
      operationType: change.operationType,
      fileId: change.documentKey._id,
      timestamp: new Date(),
      status: 'error',
      error: error.message
    });
  }

  async performFullBackup() {
    console.log('Starting full backup...');

    const files = await this.sourceDb
      .collection('production.files')
      .find()
      .toArray();

    let backedUp = 0;
    let errors = 0;

    for (const file of files) {
      try {
        await this.backupFile(file);
        backedUp++;
      } catch (error) {
        console.error(`Failed to backup ${file._id}:`, error);
        errors++;
      }
    }

    console.log(`Full backup complete: ${backedUp} files, ${errors} errors`);
    return { backedUp, errors, total: files.length };
  }

  async verifyBackup() {
    const sourceFiles = await this.sourceDb
      .collection('production.files')
      .find()
      .project({ _id: 1, length: 1 })
      .toArray();

    const backupFiles = await this.backupDb
      .collection('backup.files')
      .find()
      .project({ 'metadata.sourceId': 1, length: 1 })
      .toArray();

    const backupMap = new Map(
      backupFiles.map(f => [f.metadata.sourceId, f.length])
    );

    const missing = [];
    const sizeMismatch = [];

    for (const source of sourceFiles) {
      const sourceId = source._id.toString();
      const backupLength = backupMap.get(sourceId);

      if (!backupLength) {
        missing.push(sourceId);
      } else if (backupLength !== source.length) {
        sizeMismatch.push({
          sourceId,
          sourceLength: source.length,
          backupLength
        });
      }
    }

    return {
      totalSource: sourceFiles.length,
      totalBackup: backupFiles.length,
      missing,
      sizeMismatch,
      valid: missing.length === 0 && sizeMismatch.length === 0
    };
  }
}

// Utilisation
const sourceClient = await MongoClient.connect('mongodb://source:27017');
const backupClient = await MongoClient.connect('mongodb://backup:27017');

const backupService = new GridFSBackupService(
  sourceClient.db('production'),
  backupClient.db('backup')
);

// Full backup initial
const fullBackupResult = await backupService.performFullBackup();
console.log('Full backup:', fullBackupResult);

// Démarrer backup incrémental
await backupService.initializeIncrementalBackup();

// Vérification périodique
setInterval(async () => {
  const verification = await backupService.verifyBackup();
  console.log('Backup verification:', verification);

  if (!verification.valid) {
    console.error('Backup integrity issues detected!');
  }
}, 3600000); // Toutes les heures
```

---

## Optimisations et performances

### 1. Taille optimale des chunks

```javascript
// Pour streaming vidéo : chunks plus larges (512 KB - 1 MB)
const videoBucket = new GridFSBucket(db, {
  bucketName: 'videos',
  chunkSizeBytes: 1024 * 1024 // 1 MB
});

// Pour images : chunks par défaut (255 KB)
const imageBucket = new GridFSBucket(db, {
  bucketName: 'images'
  // chunkSizeBytes par défaut: 255 KB
});

// Pour petits fichiers : chunks plus petits (128 KB)
const docsBucket = new GridFSBucket(db, {
  bucketName: 'docs',
  chunkSizeBytes: 128 * 1024 // 128 KB
});
```

### 2. Index personnalisés pour requêtes fréquentes

```javascript
// Index sur métadonnées
await db.collection('fs.files').createIndex({
  'metadata.userId': 1,
  uploadDate: -1
});

await db.collection('fs.files').createIndex({
  contentType: 1,
  'metadata.category': 1
});

// Index texte pour recherche
await db.collection('fs.files').createIndex({
  filename: 'text',
  'metadata.tags': 'text',
  'metadata.description': 'text'
});

// Index pour taille de fichiers
await db.collection('fs.files').createIndex({
  length: 1
});
```

### 3. Monitoring des performances

```javascript
class GridFSMonitor {
  constructor(db) {
    this.db = db;
  }

  async getStorageStatistics() {
    const [filesStats, chunksStats] = await Promise.all([
      this.db.collection('fs.files').stats(),
      this.db.collection('fs.chunks').stats()
    ]);

    return {
      files: {
        count: filesStats.count,
        size: filesStats.size,
        storageSize: filesStats.storageSize,
        avgObjSize: filesStats.avgObjSize,
        indexes: filesStats.nindexes,
        indexSize: filesStats.totalIndexSize
      },
      chunks: {
        count: chunksStats.count,
        size: chunksStats.size,
        storageSize: chunksStats.storageSize,
        avgObjSize: chunksStats.avgObjSize
      },
      totalStorageSize: filesStats.storageSize + chunksStats.storageSize
    };
  }

  async getPerformanceMetrics() {
    // Temps moyen de recherche
    const searchStart = Date.now();
    await this.db.collection('fs.files')
      .find({ length: { $gt: 1000000 } })
      .limit(10)
      .toArray();
    const searchTime = Date.now() - searchStart;

    // Distribution par taille
    const sizeDistribution = await this.db.collection('fs.files').aggregate([
      {
        $bucket: {
          groupBy: '$length',
          boundaries: [0, 100000, 1000000, 10000000, 100000000, Infinity],
          default: 'Other',
          output: {
            count: { $sum: 1 },
            totalSize: { $sum: '$length' }
          }
        }
      }
    ]).toArray();

    return {
      searchTime,
      sizeDistribution
    };
  }

  async identifyLargeFiles(limit = 10) {
    return await this.db.collection('fs.files')
      .find()
      .sort({ length: -1 })
      .limit(limit)
      .project({
        filename: 1,
        length: 1,
        uploadDate: 1,
        contentType: 1
      })
      .toArray();
  }

  async identifyOrphanedChunks() {
    // Chunks sans fichier parent
    const orphaned = await this.db.collection('fs.chunks').aggregate([
      {
        $lookup: {
          from: 'fs.files',
          localField: 'files_id',
          foreignField: '_id',
          as: 'parent'
        }
      },
      {
        $match: { parent: { $size: 0 } }
      },
      {
        $group: {
          _id: '$files_id',
          chunkCount: { $sum: 1 }
        }
      }
    ]).toArray();

    return orphaned;
  }

  async cleanupOrphanedChunks() {
    const orphaned = await this.identifyOrphanedChunks();

    let deletedCount = 0;
    for (const orphan of orphaned) {
      const result = await this.db.collection('fs.chunks').deleteMany({
        files_id: orphan._id
      });
      deletedCount += result.deletedCount;
    }

    console.log(`Cleaned up ${deletedCount} orphaned chunks`);
    return deletedCount;
  }
}

// Utilisation
const monitor = new GridFSMonitor(db);

// Statistiques
const stats = await monitor.getStorageStatistics();
console.log('Storage:', stats);

// Métriques performance
const metrics = await monitor.getPerformanceMetrics();
console.log('Performance:', metrics);

// Plus gros fichiers
const largeFiles = await monitor.identifyLargeFiles();
console.log('Largest files:', largeFiles);

// Nettoyage
const cleaned = await monitor.cleanupOrphanedChunks();
```

---

## GridFS vs Alternatives

### Comparaison avec stockage cloud (S3, GCS, Azure Blob)

| Critère | GridFS | S3/GCS/Blob |
|---------|--------|-------------|
| **Intégration** | Native MongoDB | API externe |
| **Requêtes métadonnées** | ✅ MongoDB queries | ❌ API limitées |
| **Transactions** | ✅ Supportées | ❌ Non |
| **Coût** | Inclus dans MongoDB | Facturation séparée |
| **Scalabilité** | Limitée par cluster | Illimitée |
| **Performances** | Bonnes (<100MB) | Excellentes (tous fichiers) |
| **CDN** | Nécessite config | ✅ Intégré (CloudFront, etc.) |
| **Durabilité** | Dépend du cluster | 99.999999999% |

### Quand utiliser GridFS ?

✅ **Utiliser GridFS quand :**
- Fichiers < 100 MB majoritairement
- Requêtes complexes sur métadonnées nécessaires
- Transactions sur fichiers + données requises
- Infrastructure MongoDB déjà en place
- Coût d'infrastructure cloud à minimiser
- Besoin de versioning intégré

❌ **Éviter GridFS quand :**
- Fichiers > 500 MB fréquents
- Très haute volumétrie (To par jour)
- CDN global requis immédiatement
- Streaming haute performance critique
- Besoin de fonctionnalités cloud avancées (lifecycle policies, etc.)

### Pattern hybride recommandé

```javascript
class HybridStorageService {
  constructor(db, s3Client) {
    this.db = db;
    this.s3 = s3Client;
    this.gridfs = new GridFSBucket(db);
    this.sizeThreshold = 50 * 1024 * 1024; // 50 MB
  }

  async upload(filePath, metadata = {}) {
    const stats = await fs.promises.stat(filePath);

    if (stats.size < this.sizeThreshold) {
      // Petits fichiers : GridFS
      return await this.uploadToGridFS(filePath, metadata);
    } else {
      // Gros fichiers : S3
      return await this.uploadToS3(filePath, metadata);
    }
  }

  async uploadToGridFS(filePath, metadata) {
    const filename = path.basename(filePath);
    const uploadStream = this.gridfs.openUploadStream(filename, {
      metadata: {
        ...metadata,
        storage: 'gridfs'
      }
    });

    const readStream = fs.createReadStream(filePath);

    return new Promise((resolve, reject) => {
      readStream.pipe(uploadStream)
        .on('error', reject)
        .on('finish', () => resolve({
          fileId: uploadStream.id,
          storage: 'gridfs'
        }));
    });
  }

  async uploadToS3(filePath, metadata) {
    const filename = path.basename(filePath);
    const fileStream = fs.createReadStream(filePath);

    const uploadParams = {
      Bucket: 'my-bucket',
      Key: filename,
      Body: fileStream,
      Metadata: metadata
    };

    const result = await this.s3.upload(uploadParams).promise();

    // Sauvegarder référence dans MongoDB
    const fileDoc = await this.db.collection('files').insertOne({
      filename,
      s3Key: result.Key,
      s3Bucket: result.Bucket,
      s3Location: result.Location,
      storage: 's3',
      uploadDate: new Date(),
      metadata
    });

    return {
      fileId: fileDoc.insertedId,
      storage: 's3',
      url: result.Location
    };
  }

  async download(fileId) {
    // Déterminer le storage
    const fileDoc = await this.db.collection('fs.files').findOne({ _id: fileId });

    if (fileDoc) {
      // GridFS
      return await this.downloadFromGridFS(fileId);
    } else {
      // S3
      const s3Doc = await this.db.collection('files').findOne({ _id: fileId });
      if (s3Doc) {
        return await this.downloadFromS3(s3Doc.s3Key);
      }
    }

    throw new Error('File not found');
  }

  async downloadFromGridFS(fileId) {
    const downloadStream = this.gridfs.openDownloadStream(fileId);
    const chunks = [];

    return new Promise((resolve, reject) => {
      downloadStream
        .on('data', (chunk) => chunks.push(chunk))
        .on('error', reject)
        .on('end', () => resolve(Buffer.concat(chunks)));
    });
  }

  async downloadFromS3(s3Key) {
    const params = {
      Bucket: 'my-bucket',
      Key: s3Key
    };

    const data = await this.s3.getObject(params).promise();
    return data.Body;
  }
}
```

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Créer des index personnalisés sur métadonnées
await db.collection('fs.files').createIndex({
  'metadata.userId': 1,
  uploadDate: -1
});

// 2. Configurer la taille des chunks selon l'usage
const videoBucket = new GridFSBucket(db, {
  chunkSizeBytes: 1024 * 1024 // 1 MB pour vidéos
});

// 3. Toujours gérer les erreurs de stream
uploadStream.on('error', (error) => {
  console.error('Upload failed:', error);
  // Cleanup et retry
});

// 4. Implémenter une stratégie de nettoyage
async function cleanupOldFiles() {
  const threshold = new Date();
  threshold.setDate(threshold.getDate() - 90);

  const oldFiles = await db.collection('fs.files')
    .find({ uploadDate: { $lt: threshold } })
    .toArray();

  for (const file of oldFiles) {
    await bucket.delete(file._id);
  }
}

// 5. Monitorer l'utilisation du stockage
setInterval(async () => {
  const stats = await db.collection('fs.files').stats();
  console.log('GridFS storage:', stats.storageSize);
}, 3600000);
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas stocker des fichiers énormes (>500MB)
// ❌ Performance dégradée
await bucket.uploadFromStream('huge-file.iso');

// 2. Ne pas oublier de supprimer les chunks
// ❌ MAUVAIS : supprime seulement fs.files
await db.collection('fs.files').deleteOne({ _id: fileId });
// ✅ BON : utiliser bucket.delete()
await bucket.delete(fileId);

// 3. Ne pas charger tout le fichier en mémoire si pas nécessaire
// ❌ MAUVAIS pour gros fichiers
const buffer = await downloadToBuffer(fileId);
// ✅ BON : utiliser des streams
downloadStream.pipe(response);

// 4. Ne pas utiliser GridFS pour nombreux petits fichiers (<1KB)
// ❌ Overhead important
// ✅ Stocker directement dans documents MongoDB

// 5. Ne pas oublier la validation
// ❌ Accepter n'importe quel fichier
// ✅ Valider type, taille, contenu
if (stats.size > MAX_FILE_SIZE) {
  throw new Error('File too large');
}
```

---

## Conclusion

GridFS est une solution puissante pour le stockage de fichiers volumineux dans MongoDB, particulièrement adaptée aux cas où :
- ✅ L'intégration native avec MongoDB est cruciale
- ✅ Les requêtes complexes sur métadonnées sont nécessaires
- ✅ Les transactions impliquant fichiers et données sont requises
- ✅ La taille des fichiers reste raisonnable (< 100 MB généralement)

**Points clés à retenir :**
1. Comprendre l'architecture interne (fs.files + fs.chunks)
2. Configurer la taille des chunks selon le cas d'usage
3. Utiliser des streams pour éviter les problèmes de mémoire
4. Implémenter des index personnalisés sur métadonnées
5. Monitorer l'utilisation du stockage
6. Considérer une approche hybride (GridFS + cloud storage)

**Alternatives à considérer :**
- **S3/GCS/Azure Blob** : Pour très gros fichiers ou volumétrie massive
- **CDN** : Pour distribution mondiale haute performance
- **Hybrid** : GridFS pour petits/moyens + cloud pour gros fichiers

---


⏭️ [Capped Collections](/16-fonctionnalites-avancees/03-capped-collections.md)
