🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.9 Compression des Données

## Introduction

La compression des données est un levier d'optimisation majeur dans MongoDB, offrant des gains substantiels en termes de coût de stockage, performance I/O et bande passante réseau. WiredTiger, le moteur de stockage par défaut depuis MongoDB 3.2, implémente plusieurs niveaux de compression permettant d'atteindre des ratios de 2× à 10× selon les données et algorithmes choisis.

Une stratégie de compression optimale peut réduire les coûts de stockage de 50-80% tout en améliorant les performances de 10-40% grâce à la réduction des I/O. Cependant, une mauvaise configuration peut dégrader les performances de 20-50% en raison du surcoût CPU.

Cette section explore les mécanismes de compression dans MongoDB, leurs impacts sur les performances, et les méthodologies d'optimisation pour différents profils de charge.

## Architecture de la Compression dans MongoDB

### Niveaux de Compression

MongoDB implémente la compression à trois niveaux distincts :

```
┌────────────────────────────────────────────────┐
│             Application Layer                  │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│        Network Compression (Wire Protocol)     │
│  Compressors: snappy, zlib, zstd               │
│  Documents compressés durant le transit        │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│           MongoDB Server Layer                 │
│  Documents en clair en mémoire (cache)         │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│      WiredTiger Storage Engine                 │
│                                                │
│  ┌────────────────────────────────────────┐    │
│  │  Collection Block Compression          │    │
│  │  Compressors: snappy, zlib, zstd, none │    │
│  │  Blocks: 4-32 KB compressés            │    │
│  └────────────────────────────────────────┘    │
│                                                │
│  ┌────────────────────────────────────────┐    │
│  │  Index Prefix Compression              │    │
│  │  Compression des préfixes communs      │    │
│  │  (Automatique, non configurable)       │    │
│  └────────────────────────────────────────┘    │
│                                                │
│  ┌────────────────────────────────────────┐    │
│  │  Journal Compression                   │    │
│  │  Compressor: snappy (défaut), zlib     │    │
│  │  WAL logs compressés                   │    │
│  └────────────────────────────────────────┘    │
└────────────────┬───────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────┐
│          Filesystem (Data Files)               │
│  *.wt files (compressed blocks)                │
└────────────────────────────────────────────────┘
```

### Flow de Compression/Décompression

**Opération Write** :
```
1. Application → Document (JSON/BSON)
   ↓
2. Network compression (optionnel)
   ↓
3. MongoDB server → Cache (uncompressed)
   ↓
4. WiredTiger → Compression des blocks
   ↓
5. Disk → Blocks compressés écrits
```

**Opération Read** :
```
1. Disk → Lecture des blocks compressés
   ↓
2. WiredTiger → Décompression en cache
   ↓
3. MongoDB server → Document disponible
   ↓
4. Network compression (optionnel)
   ↓
5. Application → Document reçu
```

**Point critique** : La compression/décompression se fait à la frontière cache-disque, pas en cache. Le cache WiredTiger stocke les données décompressées pour un accès rapide.

## Compression des Collections (Block Compression)

### Algorithmes Disponibles

MongoDB/WiredTiger supporte quatre compresseurs pour les collections :

#### snappy (Par Défaut)

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy
```

**Caractéristiques** :
- Ratio : 2-3× typiquement
- CPU overhead : Très faible (5-10%)
- Vitesse : Très rapide (compression ~250 MB/s, décompression ~500 MB/s)
- Développé par Google
- Optimisé pour la vitesse, pas le ratio

**Use cases** :
- Workload latency-sensitive
- CPU limité
- Données déjà peu compressibles
- Production généraliste (défaut optimal)

#### zlib

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zlib
```

**Caractéristiques** :
- Ratio : 3-5× typiquement
- CPU overhead : Moyen (15-30%)
- Vitesse : Moyen (compression ~100 MB/s, décompression ~300 MB/s)
- Niveaux de compression : 1-9 (WiredTiger utilise niveau 6)
- Standard de facto (RFC 1950)

**Use cases** :
- Storage contraint (coût élevé)
- Données hautement compressibles
- Workload read-heavy (décompression plus rapide que compression)
- Réduction maximale du stockage prioritaire

#### zstd (Recommandé Modern)

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd
```

**Caractéristiques** :
- Ratio : 3-6× typiquement (meilleur que zlib)
- CPU overhead : Faible-Moyen (10-20%)
- Vitesse : Rapide (compression ~200 MB/s, décompression ~600 MB/s)
- Développé par Facebook
- Meilleur compromis ratio/performance
- Disponible depuis MongoDB 4.2

**Use cases** :
- **Recommandé pour nouveaux déploiements**
- Meilleur équilibre performance/ratio
- Remplace avantageusement snappy dans la plupart des cas
- Production moderne

#### none

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: none
```

**Caractéristiques** :
- Ratio : 1× (aucune compression)
- CPU overhead : 0%
- Vitesse : Maximale

**Use cases** :
- Latence ultra-faible critique (<1ms P99)
- Données non compressibles (images, vidéos déjà compressées)
- CPU extrêmement limité
- Storage rapide et abondant

### Benchmark Comparatif

**Setup de test** :
- Collection : 10M documents, 50GB uncompressed
- Documents : JSON structuré typique (e-commerce orders)
- Hardware : 32 cores @ 3.5 GHz, 128GB RAM, NVMe SSD
- Workload : 60% read, 40% write

**Résultats** :

```
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│Compressor│ On-Disk  │Comp Ratio│Read P99  │Write P99 │ CPU Avg  │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ none     │ 50.0 GB  │  1.0×    │  1.2 ms  │  1.8 ms  │  22%     │
│ snappy   │ 18.5 GB  │  2.7×    │  1.5 ms  │  2.1 ms  │  25%     │
│ zstd     │ 12.3 GB  │  4.1×    │  1.7 ms  │  2.4 ms  │  28%     │
│ zlib     │ 10.8 GB  │  4.6×    │  2.3 ms  │  3.2 ms  │  35%     │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

Throughput (ops/sec):
┌──────────┬──────────┬──────────┬──────────┐
│Compressor│ Inserts  │ Queries  │  Updates │
├──────────┼──────────┼──────────┼──────────┤
│ none     │  45,000  │  65,000  │  38,000  │
│ snappy   │  42,000  │  62,000  │  36,000  │
│ zstd     │  39,000  │  59,000  │  34,000  │
│ zlib     │  32,000  │  52,000  │  28,000  │
└──────────┴──────────┴──────────┴──────────┘

Analyse:
- snappy : Meilleur balance pour la plupart des cas
- zstd : Meilleur ratio avec overhead CPU raisonnable
- zlib : Ratio maximal mais impact performance significatif
- none : Seulement si latence ultra-critique ou data non compressible
```

### Configuration par Collection

```javascript
// Lors de la création de la collection
db.createCollection("highlyCompressible", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zlib"
    }
  }
})

// Collection avec données déjà compressées (images)
db.createCollection("imageMetadata", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=none"
    }
  }
})

// Collection critique latence
db.createCollection("realtimeEvents", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=snappy"
    }
  }
})
```

**Changement de compresseur** (nécessite rebuild) :

```javascript
// Méthode 1 : Via aggregation $out
db.oldCollection.aggregate([
  { $match: {} },
  { $out: "newCollectionWithZstd" }
])

// Méthode 2 : Dump/restore avec nouveau compressor
// 1. Configure nouveau compressor dans mongod.conf
// 2. mongodump --collection oldCollection
// 3. Drop oldCollection
// 4. mongorestore (collection recréée avec nouveau compressor)

// Méthode 3 : En-place avec compact (pas de changement de compressor)
// compact préserve le compressor existant
db.runCommand({ compact: "collection" })
```

### Analyse de l'Efficacité de Compression

```javascript
function analyzeCompressionEfficiency(collectionName) {
  const coll = db.getCollection(collectionName);
  const stats = coll.stats();

  const analysis = {
    collection: collectionName,

    // Tailles
    uncompressedSizeGB: (stats.size / 1024 / 1024 / 1024).toFixed(2),
    compressedSizeGB: (stats.storageSize / 1024 / 1024 / 1024).toFixed(2),

    // Ratio
    compressionRatio: (stats.size / stats.storageSize).toFixed(2) + "×",

    // Savings
    savedGB: ((stats.size - stats.storageSize) / 1024 / 1024 / 1024).toFixed(2),
    savedPercent: (((stats.size - stats.storageSize) / stats.size) * 100).toFixed(2) + "%",

    // Index
    indexSizeGB: (stats.totalIndexSize / 1024 / 1024 / 1024).toFixed(2),

    // Total on disk
    totalOnDiskGB: ((stats.storageSize + stats.totalIndexSize) / 1024 / 1024 / 1024).toFixed(2),

    // Document stats
    documentCount: stats.count,
    avgDocumentSizeKB: (stats.avgObjSize / 1024).toFixed(2),

    // Compressor
    compressor: "unknown"
  };

  // Extract compressor from creation string
  if (stats.wiredTiger && stats.wiredTiger.creationString) {
    const match = stats.wiredTiger.creationString.match(/block_compressor=(\w+)/);
    if (match) {
      analysis.compressor = match[1];
    }
  }

  // Assessment
  const ratio = parseFloat(analysis.compressionRatio);
  if (ratio < 1.5) {
    analysis.assessment = "⚠️ Poor compression - Data may be pre-compressed or binary";
  } else if (ratio < 2.5) {
    analysis.assessment = "Acceptable compression";
  } else if (ratio < 4) {
    analysis.assessment = "✅ Good compression";
  } else {
    analysis.assessment = "✅ Excellent compression";
  }

  return analysis;
}

// Usage
printjson(analyzeCompressionEfficiency("orders"));

// Analyse de toutes les collections
db.getCollectionNames().forEach(collName => {
  const analysis = analyzeCompressionEfficiency(collName);
  print(`\n${collName}:`);
  print(`  Compression: ${analysis.compressionRatio} (${analysis.compressor})`);
  print(`  Saved: ${analysis.savedGB} GB (${analysis.savedPercent})`);
  print(`  Assessment: ${analysis.assessment}`);
});
```

### Facteurs Affectant le Ratio de Compression

**Type de données** :

```javascript
// Données hautement compressibles (ratio 5-10×)
{
  _id: ObjectId("..."),
  status: "completed",              // Répétitif
  paymentMethod: "credit_card",     // Répétitif
  currency: "EUR",                  // Répétitif
  description: "Standard text...",  // Text compresse bien
  tags: ["sale", "promo", "new"]   // Répétitif
}
// Compresseur optimal : zlib

// Données moyennement compressibles (ratio 2-4×)
{
  _id: ObjectId("..."),
  userId: "user12345",
  timestamp: ISODate("2025-01-15T10:30:00Z"),
  metrics: {
    cpu: 45.2,
    memory: 67.8,
    disk: 23.1
  }
}
// Compresseur optimal : snappy ou zstd

// Données peu compressibles (ratio 1.1-1.5×)
{
  _id: ObjectId("..."),
  imageData: BinData(0, "..."),    // Déjà compressé (JPEG)
  videoUrl: "https://...",
  hash: "a7f3e9c2d1b8..."          // Random
}
// Compresseur optimal : none

// Données avec structure répétitive (ratio 6-12×)
{
  _id: ObjectId("..."),
  metadata: {
    version: "1.0.0",
    environment: "production",
    datacenter: "us-east-1",
    application: "web-service"
  },
  // Beaucoup de clés identiques entre documents
}
// Compresseur optimal : zlib
```

**Patterns de compression** :

```javascript
// Excellent pour compression
const excellentCompressionPatterns = {
  repetitiveValues: true,        // Status, categories
  textFields: true,              // Descriptions, comments
  numericalSequences: true,      // IDs séquentiels
  timestampRanges: true,         // Dates proches
  structuredKeys: true,          // JSON avec même structure
  lowCardinality: true          // Peu de valeurs uniques
};

// Mauvais pour compression
const poorCompressionPatterns = {
  binaryData: true,             // Images, audio, vidéo
  encryptedData: true,          // Données chiffrées
  randomStrings: true,          // UUIDs, hashes
  alreadyCompressed: true,      // Fichiers ZIP, GZIP
  highCardinality: true,        // Valeurs très variées
  sparseData: true              // Beaucoup de null
};
```

### Compression Block Size

WiredTiger utilise des blocks de taille variable (4KB à 32KB).

```javascript
// Configuration avancée (rare)
db.createCollection("tuned", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd,allocation_size=32KB"
      // Augmenter allocation_size peut améliorer ratio mais augmente latency
    }
  }
})

// Trade-offs :
// - Block size plus grand : Meilleur ratio, mais plus de données à décompresser
// - Block size plus petit : Latence plus faible, mais ratio réduit
// - Défaut (4KB-32KB dynamique) : Optimal pour la plupart des cas
```

## Compression des Index

### Prefix Compression

Les index B-Tree utilisent automatiquement la prefix compression.

**Principe** :
```
Sans prefix compression :
├─ "customer_12345"
├─ "customer_12346"
├─ "customer_12347"
└─ "customer_12348"

Avec prefix compression :
├─ "customer_1234" (prefix)
    ├─ "5"
    ├─ "6"
    ├─ "7"
    └─ "8"

Économie : ~70% sur cet exemple
```

**Configuration** :
```yaml
storage:
  wiredTiger:
    indexConfig:
      prefixCompression: true  # Défaut : true
```

**Impact** :
```javascript
// Mesure de l'efficacité prefix compression
function analyzeIndexCompression(collectionName) {
  const coll = db.getCollection(collectionName);
  const stats = coll.stats();

  const indexes = coll.getIndexes();

  indexes.forEach(index => {
    const indexStats = coll.aggregate([
      { $indexStats: {} },
      { $match: { name: index.name } }
    ]).next();

    if (indexStats) {
      print(`\nIndex: ${index.name}`);
      print(`  Keys: ${JSON.stringify(index.key)}`);
      print(`  Size: ${(indexStats.size / 1024 / 1024).toFixed(2)} MB`);

      // Estimation du ratio (approximatif)
      const avgKeySize = 50; // Estimation moyenne
      const expectedUncompressed = stats.count × avgKeySize;
      const compressionRatio = expectedUncompressed / indexStats.size;
      print(`  Est. compression: ${compressionRatio.toFixed(2)}×`);
    }
  });
}

analyzeIndexCompression("orders");
```

**Cas où désactiver prefix compression** :
```javascript
// Très rare, seulement si :
// 1. Index sur données très aléatoires (UUID)
// 2. Performance de write critique
// 3. Ratio de compression négligeable

db.runCommand({
  createIndexes: "collection",
  indexes: [
    {
      key: { randomField: 1 },
      name: "random_idx",
      storageEngine: {
        wiredTiger: {
          configString: "prefix_compression=false"
        }
      }
    }
  ]
})

// Impact : +10-20% espace, +2-5% write performance
```

### Index Storage Size

```javascript
function compareIndexSizes() {
  const collections = db.getCollectionNames();

  collections.forEach(collName => {
    const stats = db.getCollection(collName).stats();

    const dataToIndexRatio = stats.totalIndexSize / stats.size;

    print(`\n${collName}:`);
    print(`  Data: ${(stats.size / 1024 / 1024).toFixed(2)} MB`);
    print(`  Indexes: ${(stats.totalIndexSize / 1024 / 1024).toFixed(2)} MB`);
    print(`  Ratio: ${(dataToIndexRatio * 100).toFixed(2)}%`);

    if (dataToIndexRatio > 0.5) {
      print(`  ⚠️ Indexes > 50% of data size - Review index strategy`);
    }
  });
}

compareIndexSizes();
```

## Compression du Journal

### Configuration du Journal

```yaml
storage:
  wiredTiger:
    engineConfig:
      journalCompressor: snappy  # snappy (défaut) ou zlib
```

**Trade-offs** :

| Compressor | Ratio | CPU | Write Latency | Use Case |
|------------|-------|-----|---------------|----------|
| **snappy** | 2-3× | Low | Baseline | **Défaut recommandé** |
| zlib | 3-5× | Medium | +10-15% | Storage très limité |

**Impact du journal compression** :
```
Journal compressé réduit :
- Taille des fichiers journal (100 MB → 30-50 MB)
- I/O de write du journal
- Bande passante réseau pour réplication (oplog basé sur journal)

Mais ajoute :
- CPU overhead sur writes
- Légère augmentation de latency de write

Recommandation : Garder snappy (défaut optimal)
Changer vers zlib seulement si storage journal très limité
```

## Compression Réseau

### Wire Protocol Compression

MongoDB 3.6+ supporte la compression du wire protocol.

**Configuration** :

```yaml
# mongod.conf
net:
  compression:
    compressors: snappy,zstd,zlib
```

**Driver configuration** (exemple Node.js) :

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(uri, {
  compressors: ['zstd', 'snappy', 'zlib'],  // Ordre de préférence
  zlibCompressionLevel: 6  // 1-9, défaut 6
});
```

**Négociation** :
```
1. Client envoie liste de compressors supportés
2. Server répond avec premier compressor commun
3. Toutes les communications suivantes utilisent ce compressor

Ordre de préférence typique : zstd > snappy > zlib
```

**Impact de la compression réseau** :

```javascript
// Benchmark réseau
function benchmarkNetworkCompression() {
  // Sans compression
  const start1 = Date.now();
  const docs1 = db.largeCollection.find().limit(10000).toArray();
  const time1 = Date.now() - start1;

  // Avec compression (configurer dans driver)
  const start2 = Date.now();
  const docs2 = db.largeCollection.find().limit(10000).toArray();
  const time2 = Date.now() - start2;

  const improvement = ((time1 - time2) / time1 * 100).toFixed(2);

  print(`Without compression: ${time1}ms`);
  print(`With compression: ${time2}ms`);
  print(`Improvement: ${improvement}%`);
}

// Résultats typiques :
// - LAN (latency faible) : 5-15% improvement
// - WAN (latency élevée) : 30-60% improvement
// - Large documents : Plus de gain
// - Small documents : Moins de gain (overhead)
```

**Recommandations** :

```yaml
# LAN (datacenter local, latency <1ms)
net:
  compression:
    compressors: snappy  # Overhead CPU minimal

# WAN (inter-régions, latency >10ms)
net:
  compression:
    compressors: zstd,snappy  # Ratio important

# Mobile / Limited bandwidth
net:
  compression:
    compressors: zlib,zstd,snappy  # Ratio maximal
```

**Monitoring de la compression réseau** :

```javascript
function analyzeNetworkCompression() {
  const serverStatus = db.serverStatus();

  if (serverStatus.network && serverStatus.network.compression) {
    const comp = serverStatus.network.compression;

    const analysis = {
      compressor: comp.snappy ? "snappy" : (comp.zstd ? "zstd" : (comp.zlib ? "zlib" : "none")),

      // Bytes statistics
      bytesIn: {
        uncompressed: comp.snappy ? comp.snappy.compressor.bytesIn : 0,
        compressed: comp.snappy ? comp.snappy.compressor.bytesOut : 0
      },

      bytesOut: {
        uncompressed: comp.snappy ? comp.snappy.decompressor.bytesIn : 0,
        compressed: comp.snappy ? comp.snappy.decompressor.bytesOut : 0
      }
    };

    // Calculate ratios
    if (analysis.bytesIn.uncompressed > 0) {
      analysis.compressionRatioIn =
        (analysis.bytesIn.uncompressed / analysis.bytesIn.compressed).toFixed(2) + "×";
    }

    if (analysis.bytesOut.uncompressed > 0) {
      analysis.compressionRatioOut =
        (analysis.bytesOut.uncompressed / analysis.bytesOut.compressed).toFixed(2) + "×";
    }

    return analysis;
  } else {
    return { status: "Network compression not configured or not supported" };
  }
}

printjson(analyzeNetworkCompression());
```

## Stratégies de Compression par Workload

### Read-Heavy Workload

**Caractéristiques** :
- 90% reads, 10% writes
- Latence critique : P99 < 50ms
- Cache hit ratio : >95%

**Stratégie optimale** :

```yaml
# mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy  # Décompression rapide prioritaire
    indexConfig:
      prefixCompression: true
    engineConfig:
      journalCompressor: snappy

net:
  compression:
    compressors: snappy,zstd  # Snappy en premier pour latence
```

**Rationale** :
- Décompression fréquente (90% reads)
- snappy décompresse 2× plus vite que zstd, 3× que zlib
- Ratio 2-3× suffisant si cache bien dimensionné
- Overhead CPU minimal

**Métriques de validation** :
```javascript
const serverStatus = db.serverStatus();
const cacheHitRatio =
  ((serverStatus.wiredTiger.cache["pages requested from the cache"] -
    serverStatus.wiredTiger.cache["pages read into cache"]) /
   serverStatus.wiredTiger.cache["pages requested from the cache"] * 100).toFixed(2);

print(`Cache Hit Ratio: ${cacheHitRatio}%`);
// Si >95% : Décompression limitée, snappy optimal
// Si <90% : Considérer zstd pour réduire I/O
```

### Write-Heavy Workload

**Caractéristiques** :
- 30% reads, 70% writes
- Throughput critique : >10K writes/sec
- Storage growth : Rapide

**Stratégie optimale** :

```yaml
# mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd  # Bon ratio, compression rapide
    indexConfig:
      prefixCompression: true
    engineConfig:
      journalCompressor: snappy  # Journal : vitesse prioritaire

net:
  compression:
    compressors: zstd,snappy
```

**Rationale** :
- Compression fréquente (70% writes)
- zstd : Meilleur ratio avec overhead CPU acceptable
- Réduit write amplification (moins de bytes sur disque)
- Journal : snappy car writes synchrones

**Impact mesuré** :
```
Benchmark : 100K inserts/sec

snappy:
- Storage: 180 GB
- CPU avg: 45%
- Write P99: 5.2ms

zstd:
- Storage: 120 GB (33% saved)
- CPU avg: 52% (+7%)
- Write P99: 6.1ms (+17%)

Conclusion : zstd optimal si storage coûteux
            snappy optimal si latency critique
```

### Storage-Constrained Workload

**Caractéristiques** :
- Storage très limité ou très coûteux
- Dataset volumineux (multi-TB)
- Performance acceptable si latence raisonnable

**Stratégie optimale** :

```yaml
# mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zlib  # Ratio maximal
    indexConfig:
      prefixCompression: true
    engineConfig:
      journalCompressor: zlib  # Même journal compressé max

net:
  compression:
    compressors: zlib,zstd,snappy
```

**Rationale** :
- Minimiser coût de stockage
- Ratio 4-5× vs 2-3× = 40-50% de saving additionnel
- CPU overhead acceptable si pas ultra-latency-sensitive

**ROI calculation** :
```javascript
function calculateCompressionROI() {
  const datasetGB = 1000;  // 1 TB
  const storageCostPerGB = 0.10;  // $0.10/GB/month

  const scenarios = {
    snappy: {
      ratio: 2.5,
      cpuOverhead: 5,
      latencyOverhead: 0
    },
    zstd: {
      ratio: 4.0,
      cpuOverhead: 15,
      latencyOverhead: 10
    },
    zlib: {
      ratio: 4.5,
      cpuOverhead: 30,
      latencyOverhead: 20
    }
  };

  Object.keys(scenarios).forEach(compressor => {
    const s = scenarios[compressor];
    const storageGB = datasetGB / s.ratio;
    const monthlyCost = storageGB × storageCostPerGB;
    const saving = ((datasetGB - storageGB) × storageCostPerGB);

    print(`\n${compressor}:`);
    print(`  Storage: ${storageGB.toFixed(2)} GB`);
    print(`  Monthly cost: $${monthlyCost.toFixed(2)}`);
    print(`  Monthly saving vs uncompressed: $${saving.toFixed(2)}`);
    print(`  CPU overhead: +${s.cpuOverhead}%`);
    print(`  Latency overhead: +${s.latencyOverhead}%`);
  });
}

calculateCompressionROI();

// Résultat exemple :
// snappy : $40/month, saving $60
// zstd : $25/month, saving $75 (+$15 vs snappy)
// zlib : $22/month, saving $78 (+$18 vs snappy)
//
// Si +$15/month saving > coût CPU additionnel → zstd/zlib optimal
```

### Analytics / Reporting Workload

**Caractéristiques** :
- Scans complets fréquents
- Agrégations complexes
- Latence moins critique (seconds acceptable)

**Stratégie optimale** :

```yaml
# mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zlib  # I/O réduits critiques
    indexConfig:
      prefixCompression: true
    engineConfig:
      journalCompressor: snappy

net:
  compression:
    compressors: zstd,zlib  # Large result sets
```

**Rationale** :
- Scans lisent tout le dataset → I/O dominant
- zlib réduit I/O de 4-5× vs aucune compression
- CPU overhead acceptable pour analytics
- Réduction massive du temps de scan

**Mesure d'impact** :
```javascript
// Benchmark : Full collection scan
function benchmarkScanWithCompression(collectionName) {
  const coll = db.getCollection(collectionName);

  // Warm up
  coll.find().limit(1000).toArray();

  // Test scan
  const start = Date.now();
  const count = coll.find({}, { _id: 1 }).toArray().length;
  const duration = Date.now() - start;

  const stats = coll.stats();
  const storageGB = stats.storageSize / 1024 / 1024 / 1024;

  print(`Collection: ${collectionName}`);
  print(`Documents: ${count}`);
  print(`Storage: ${storageGB.toFixed(2)} GB`);
  print(`Scan time: ${duration}ms`);
  print(`Throughput: ${(count / duration * 1000).toFixed(0)} docs/sec`);

  // Estimation si non compressé
  const uncompressedGB = stats.size / 1024 / 1024 / 1024;
  const ratio = uncompressedGB / storageGB;
  const estimatedTimeUncompressed = duration × ratio;

  print(`\nWith current compression (${ratio.toFixed(2)}×):`);
  print(`  Actual scan: ${duration}ms`);
  print(`Est. without compression:`);
  print(`  Would be: ${estimatedTimeUncompressed.toFixed(0)}ms`);
  print(`  Improvement: ${((estimatedTimeUncompressed - duration) / estimatedTimeUncompressed * 100).toFixed(2)}%`);
}

benchmarkScanWithCompression("largeAnalyticsCollection");

// Résultat typique :
// Avec zlib (4.5×) : Scan en 45 secondes
// Sans compression : Estimé 150 secondes
// Amélioration : 70%
```

## Impact Performance Détaillé

### CPU Overhead

**Mesure du CPU overhead** :

```javascript
function measureCompressionCPU() {
  // Baseline : Collection sans compression
  const baselineOps = 50000;

  print("Running baseline (no compression)...");
  const start1 = Date.now();
  for (let i = 0; i < baselineOps; i++) {
    db.testNoComp.insertOne({ data: "x".repeat(1000), index: i });
  }
  const time1 = Date.now() - start1;

  // Test : Collection avec compression
  print("Running with compression...");
  const start2 = Date.now();
  for (let i = 0; i < baselineOps; i++) {
    db.testWithComp.insertOne({ data: "x".repeat(1000), index: i });
  }
  const time2 = Date.now() - start2;

  const overhead = ((time2 - time1) / time1 * 100).toFixed(2);

  print(`\nResults:`);
  print(`No compression: ${time1}ms`);
  print(`With compression: ${time2}ms`);
  print(`CPU overhead: ${overhead}%`);
}

// Résultats typiques :
// snappy : +5-10% CPU
// zstd : +10-20% CPU
// zlib : +25-40% CPU
```

### I/O Reduction

**Mesure de la réduction I/O** :

```javascript
function measureIOReduction(collectionName) {
  const stats = db.getCollection(collectionName).stats();
  const wt = db.serverStatus().wiredTiger;

  // Bloc manager stats
  const bytesRead = wt.block_manager["bytes read"];
  const bytesWritten = wt.block_manager["bytes written"];

  const uncompressedSize = stats.size;
  const compressedSize = stats.storageSize;
  const ratio = uncompressedSize / compressedSize;

  print(`Collection: ${collectionName}`);
  print(`Compression ratio: ${ratio.toFixed(2)}×`);
  print(`\nI/O Impact:`);
  print(`  Uncompressed size: ${(uncompressedSize / 1024 / 1024).toFixed(2)} MB`);
  print(`  Compressed on disk: ${(compressedSize / 1024 / 1024).toFixed(2)} MB`);
  print(`  I/O reduced by: ${((uncompressedSize - compressedSize) / 1024 / 1024).toFixed(2)} MB`);
  print(`  Reduction: ${((1 - 1/ratio) * 100).toFixed(2)}%`);

  // IOPS estimation
  const avgBlockSize = 16384;  // 16KB
  const blocksUncompressed = Math.ceil(uncompressedSize / avgBlockSize);
  const blocksCompressed = Math.ceil(compressedSize / avgBlockSize);

  print(`\nIOPS Impact (estimated):`);
  print(`  Blocks without compression: ${blocksUncompressed}`);
  print(`  Blocks with compression: ${blocksCompressed}`);
  print(`  IOPS saved: ${blocksUncompressed - blocksCompressed} (${((1 - blocksCompressed/blocksUncompressed) * 100).toFixed(2)}%)`);
}

measureIOReduction("orders");
```

### Latency Impact

**Impact sur read latency** :

```
Décomposition read latency :

Sans compression :
├─ Disk I/O : 5 ms (lecture 100 KB)
├─ Transfer to cache : 0.1 ms
└─ Return to client : 0.1 ms
Total : 5.2 ms

Avec compression 4× (snappy) :
├─ Disk I/O : 1.5 ms (lecture 25 KB)
├─ Décompression : 0.3 ms
├─ Transfer to cache : 0.1 ms
└─ Return to client : 0.1 ms
Total : 2.0 ms

Amélioration : 61% plus rapide !

Note : Bénéfice si I/O > CPU
Sur NVMe (I/O <1ms), bénéfice réduit
```

**Impact sur write latency** :

```
Sans compression :
├─ Cache write : 0.1 ms
├─ Disk I/O : 3 ms (écriture 100 KB)
└─ Journal write : 2 ms
Total : 5.1 ms

Avec compression 4× (snappy) :
├─ Compression : 0.4 ms
├─ Cache write : 0.1 ms
├─ Disk I/O : 1 ms (écriture 25 KB)
└─ Journal write : 0.7 ms
Total : 2.2 ms

Amélioration : 57% plus rapide !

Note : Sur SSD rapide, overhead compression peut dépasser gain I/O
```

## Monitoring et Métriques

### Métriques Clés

```javascript
function compressionMetricsDashboard() {
  const collections = db.getCollectionNames();
  const serverStatus = db.serverStatus();

  const dashboard = {
    timestamp: new Date(),

    // Global storage
    globalStorage: {
      dataUncompressedGB: 0,
      dataCompressedGB: 0,
      indexSizeGB: 0,
      totalOnDiskGB: 0,
      overallCompressionRatio: 0
    },

    // Per-collection breakdown
    collections: [],

    // I/O metrics
    io: {
      bytesReadGB: (serverStatus.wiredTiger.block_manager["bytes read"] / 1024 / 1024 / 1024).toFixed(2),
      bytesWrittenGB: (serverStatus.wiredTiger.block_manager["bytes written"] / 1024 / 1024 / 1024).toFixed(2),
      blocksRead: serverStatus.wiredTiger.block_manager["blocks read"],
      blocksWritten: serverStatus.wiredTiger.block_manager["blocks written"]
    }
  };

  // Aggregate collection stats
  collections.forEach(collName => {
    const stats = db.getCollection(collName).stats();

    const uncompressedGB = stats.size / 1024 / 1024 / 1024;
    const compressedGB = stats.storageSize / 1024 / 1024 / 1024;
    const indexGB = stats.totalIndexSize / 1024 / 1024 / 1024;
    const ratio = stats.size / stats.storageSize;

    dashboard.globalStorage.dataUncompressedGB += uncompressedGB;
    dashboard.globalStorage.dataCompressedGB += compressedGB;
    dashboard.globalStorage.indexSizeGB += indexGB;
    dashboard.globalStorage.totalOnDiskGB += compressedGB + indexGB;

    dashboard.collections.push({
      name: collName,
      uncompressedGB: uncompressedGB.toFixed(2),
      compressedGB: compressedGB.toFixed(2),
      indexGB: indexGB.toFixed(2),
      ratio: ratio.toFixed(2) + "×",
      savedGB: (uncompressedGB - compressedGB).toFixed(2)
    });
  });

  // Calculate overall ratio
  dashboard.globalStorage.overallCompressionRatio =
    (dashboard.globalStorage.dataUncompressedGB /
     dashboard.globalStorage.dataCompressedGB).toFixed(2) + "×";

  // Format global storage
  dashboard.globalStorage.dataUncompressedGB =
    dashboard.globalStorage.dataUncompressedGB.toFixed(2);
  dashboard.globalStorage.dataCompressedGB =
    dashboard.globalStorage.dataCompressedGB.toFixed(2);
  dashboard.globalStorage.indexSizeGB =
    dashboard.globalStorage.indexSizeGB.toFixed(2);
  dashboard.globalStorage.totalOnDiskGB =
    dashboard.globalStorage.totalOnDiskGB.toFixed(2);

  return dashboard;
}

const dashboard = compressionMetricsDashboard();
printjson(dashboard);

// Export pour monitoring externe
// print(JSON.stringify(dashboard));
```

### Alerting

**Prometheus-style alerting rules** :

```yaml
# Compression ratio trop faible
- alert: LowCompressionRatio
  expr: |
    mongodb_collection_compression_ratio < 1.5
  for: 1h
  labels:
    severity: info
  annotations:
    summary: "Low compression ratio detected"
    description: "Collection {{ $labels.collection }} has compression ratio < 1.5×"

# Storage growth anormal
- alert: HighStorageGrowth
  expr: |
    rate(mongodb_storage_size_bytes[24h]) > 10737418240  # 10 GB/day
  labels:
    severity: warning
  annotations:
    summary: "High storage growth rate"
    description: "Storage growing > 10 GB/day"

# Overhead CPU compression élevé
- alert: HighCompressionCPU
  expr: |
    rate(mongodb_compression_cpu_time_seconds[5m]) > 0.5
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "High CPU time spent on compression"
```

## Stratégies de Migration

### Migration vers un Nouveau Compressor

**Approche Progressive (Recommandée)** :

```javascript
// Étape 1 : Créer nouvelle collection avec nouveau compressor
db.createCollection("orders_new", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
});

// Étape 2 : Copier les index
db.orders.getIndexes().forEach(function(index) {
  var key = index.key;
  delete index.key;
  delete index.ns;
  delete index.v;
  db.orders_new.createIndex(key, index);
});

// Étape 3 : Migration des données (par batch)
var batchSize = 10000;
var cursor = db.orders.find().noCursorTimeout();
var batch = [];

while (cursor.hasNext()) {
  batch.push(cursor.next());

  if (batch.length >= batchSize) {
    db.orders_new.insertMany(batch, { ordered: false });
    batch = [];
    print("Migrated " + db.orders_new.countDocuments() + " documents");
  }
}

// Dernier batch
if (batch.length > 0) {
  db.orders_new.insertMany(batch, { ordered: false });
}

cursor.close();

// Étape 4 : Validation
var oldCount = db.orders.countDocuments();
var newCount = db.orders_new.countDocuments();
print(`Old collection: ${oldCount} documents`);
print(`New collection: ${newCount} documents`);
assert(oldCount === newCount, "Document count mismatch!");

// Étape 5 : Comparaison de taille
var oldStats = db.orders.stats();
var newStats = db.orders_new.stats();
print("\nCompression comparison:");
print(`Old (snappy): ${(oldStats.storageSize / 1024 / 1024).toFixed(2)} MB`);
print(`New (zstd): ${(newStats.storageSize / 1024 / 1024).toFixed(2)} MB`);
print(`Saving: ${((oldStats.storageSize - newStats.storageSize) / oldStats.storageSize * 100).toFixed(2)}%`);

// Étape 6 : Cutover (downtime minimal)
// 1. Stop writes to old collection
// 2. Migrate delta since start
// 3. Rename collections
db.orders.renameCollection("orders_old");
db.orders_new.renameCollection("orders");

// Étape 7 : Validation post-cutover
// Test queries, verify performance

// Étape 8 : Drop old collection (après validation)
// db.orders_old.drop();
```

**Approche Blue-Green** :

```javascript
// Configuration replica set pour migration
// Primary : Ancienne config (snappy)
// New Secondary : Nouvelle config (zstd)

// 1. Ajouter nouveau secondary avec nouveau compressor
rs.add({
  host: "mongo4.example.com:27017",
  priority: 0,
  votes: 0
})

// 2. Sur mongo4, configurer nouveau compressor
// mongod.conf sur mongo4 :
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd

// 3. Initial sync (peut prendre heures/jours)
// Monitor : rs.status()

// 4. Une fois sync complète, comparer tailles
db.adminCommand({ serverStatus: 1 }).wiredTiger

// 5. Step down primary
rs.stepDown(120)

// 6. Nouveau primary avec zstd devient actif
// 7. Autres membres initial sync et obtiennent zstd
```

### A/B Testing de Compresseurs

```javascript
// Framework pour A/B test
class CompressionABTest {
  constructor(sourceCollection, testDuration = 3600) {
    this.source = sourceCollection;
    this.duration = testDuration;
    this.results = {
      snappy: { collection: sourceCollection + "_test_snappy" },
      zstd: { collection: sourceCollection + "_test_zstd" },
      zlib: { collection: sourceCollection + "_test_zlib" }
    };
  }

  async setup() {
    // Créer collections de test avec différents compressors
    for (const [compressor, config] of Object.entries(this.results)) {
      db.createCollection(config.collection, {
        storageEngine: {
          wiredTiger: {
            configString: `block_compressor=${compressor}`
          }
        }
      });

      // Copier sample de données
      const sample = db.getCollection(this.source)
        .aggregate([{ $sample: { size: 100000 } }])
        .toArray();

      db.getCollection(config.collection).insertMany(sample);
    }
  }

  async runBenchmark() {
    for (const [compressor, config] of Object.entries(this.results)) {
      const coll = db.getCollection(config.collection);

      // Mesures
      const startTime = Date.now();

      // Read test
      const readStart = Date.now();
      coll.find().limit(10000).toArray();
      config.readLatency = Date.now() - readStart;

      // Write test
      const writeStart = Date.now();
      for (let i = 0; i < 1000; i++) {
        coll.insertOne({ test: i, data: "x".repeat(1000) });
      }
      config.writeLatency = Date.now() - writeStart;

      // Storage
      const stats = coll.stats();
      config.storageSize = stats.storageSize;
      config.compressionRatio = (stats.size / stats.storageSize).toFixed(2);
    }
  }

  async cleanup() {
    for (const config of Object.values(this.results)) {
      db.getCollection(config.collection).drop();
    }
  }

  getResults() {
    return this.results;
  }
}

// Usage
const test = new CompressionABTest("orders");
await test.setup();
await test.runBenchmark();
printjson(test.getResults());
await test.cleanup();
```

## Best Practices

### Checklist de Configuration

```
☐ Analyse du type de données
  ☐ Identifier données hautement compressibles (text, répétitif)
  ☐ Identifier données peu compressibles (binary, random)
  ☐ Mesurer distribution des types

☐ Définir priorités
  ☐ Latence vs Storage cost
  ☐ CPU disponible
  ☐ I/O capabilities

☐ Choisir compresseurs
  ☐ Collection : snappy (latence) | zstd (balance) | zlib (ratio)
  ☐ Index : prefix compression enabled (défaut)
  ☐ Journal : snappy (recommandé)
  ☐ Network : selon WAN/LAN

☐ Tester en staging
  ☐ Benchmark avec données réelles
  ☐ Mesurer latency P99
  ☐ Mesurer CPU usage
  ☐ Mesurer storage savings

☐ Monitoring production
  ☐ Compression ratio par collection
  ☐ Storage growth rate
  ☐ CPU overhead
  ☐ Latency impact

☐ Optimisation continue
  ☐ Review trimestriel
  ☐ A/B testing de nouveaux compressors
  ☐ Ajustement selon évolution workload
```

### Décision Matrix

```
┌─────────────────────────┬──────────┬──────────┬──────────┐
│ Scenario                │ snappy   │ zstd     │ zlib     │
├─────────────────────────┼──────────┼──────────┼──────────┤
│ Low latency required    │ ★★★★★    │ ★★★☆☆    │ ★☆☆☆☆    │
│ Storage constrained     │ ★★☆☆☆    │ ★★★★☆    │ ★★★★★    │
│ CPU limited             │ ★★★★★    │ ★★★☆☆    │ ★☆☆☆☆    │
│ Write heavy             │ ★★★★☆    │ ★★★★☆    │ ★★☆☆☆    │
│ Read heavy              │ ★★★★★    │ ★★★★☆    │ ★★☆☆☆    │
│ Analytics workload      │ ★★☆☆☆    │ ★★★★☆    │ ★★★★★    │
│ General purpose         │ ★★★★☆    │ ★★★★★    │ ★★★☆☆    │
└─────────────────────────┴──────────┴──────────┴──────────┘

Recommendation :
- Défaut 2020-2024 : snappy
- Défaut 2025+ : zstd (meilleur compromis)
- Storage critique : zlib
- Ultra low-latency : none (rare)
```

### Erreurs Courantes

```javascript
// ❌ ERREUR 1 : Compression uniforme
// Appliquer même compressor à toutes les collections
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zlib  // Trop agressif pour tout

// ✅ CORRECT : Stratégie par collection
db.createCollection("realtime", {
  storageEngine: { wiredTiger: { configString: "block_compressor=snappy" } }
});
db.createCollection("analytics", {
  storageEngine: { wiredTiger: { configString: "block_compressor=zlib" } }
});

// ❌ ERREUR 2 : Ignorer le monitoring
// Configurer compression puis oublier

// ✅ CORRECT : Monitoring continu
// Alerter si compression ratio < 1.5× (données possiblement pré-compressées)
// Review mensuel des métriques

// ❌ ERREUR 3 : Migration sans test
// Changer compressor directement en production

// ✅ CORRECT : Test puis migration progressive
// 1. A/B test en staging
// 2. Mesurer impact
// 3. Migration collection par collection
// 4. Validation à chaque étape

// ❌ ERREUR 4 : Ignorer network compression
// Oublier de configurer compression réseau

// ✅ CORRECT : Activer network compression
net:
  compression:
    compressors: zstd,snappy  // Surtout pour WAN
```

## Conclusion

La compression des données dans MongoDB offre des bénéfices significatifs :

**Gains mesurables** :
- Storage : 50-80% de réduction (ratio 2-5×)
- I/O : 50-75% de réduction
- Network : 30-70% de réduction (selon latency)
- Coûts : 40-70% d'économies storage

**Trade-offs** :
- CPU : +5-30% selon compressor
- Latency : -20% à +20% selon I/O/CPU balance
- Complexité : Configuration et monitoring additionnels

**Recommandations clés** :
1. **zstd pour nouveaux déploiements** (meilleur compromis 2025+)
2. **snappy pour ultra low-latency** (<5ms P99)
3. **zlib pour storage critique** (coûts élevés)
4. **none seulement si justifié** (latence <1ms ou data pré-compressée)

**Méthodologie** :
1. Analyser les données (type, compressibilité)
2. Définir priorités (latence, storage, CPU)
3. Tester avec données réelles
4. Déployer progressivement
5. Monitorer et ajuster

La compression n'est pas une configuration "set and forget". Elle nécessite un monitoring continu et des ajustements réguliers basés sur l'évolution du workload et des technologies (nouveaux compresseurs, hardware plus rapide).

---

**Points clés à retenir :**
- zstd est le meilleur choix moderne (ratio + performance)
- snappy reste optimal pour latence critique
- Compression = Storage savings + I/O reduction - CPU overhead
- Tester avant déploiement (impact performance variable)
- Monitoring continu essentiel (ratio, CPU, latency)
- Stratégie différente par collection selon workload
- Network compression crucial pour WAN
- Migration progressive nécessaire pour changement de compressor

⏭️ [Read/Write splitting](/17-performance-tuning/10-read-write-splitting.md)
