🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.5 Synchronisation Bidirectionnelle

## Introduction

La synchronisation bidirectionnelle représente l'un des défis les plus complexes en migration de données. Contrairement à la synchronisation unidirectionnelle (CDC simple), elle permet des modifications concurrentes dans deux systèmes hétérogènes (SQL et MongoDB) avec réconciliation automatique des conflits. Cette approche est essentielle pour les migrations incrémentales nécessitant une période de coexistence prolongée avec zéro perte de données.

Cette section explore les architectures, algorithmes et patterns éprouvés pour implémenter une synchronisation bidirectionnelle fiable à l'échelle production.

---

## 🎯 Principes et Défis

### Le problème fondamental

**Scénario typique**
```
T0 : [Record A v1] existe dans SQL et MongoDB (sync)

T1 : User1 modifie A → v2 dans SQL
T1 : User2 modifie A → v2' dans MongoDB (concurrent)

T2 : Sync SQL → MongoDB détecte conflit (v2 vs v2')
T2 : Sync MongoDB → SQL détecte conflit (v2' vs v2)

Question : Quelle version prévaloir ? Comment réconcilier ?
```

### Propriétés désirées (théorème CAP)

**Consistency (Cohérence)**
- Les deux systèmes doivent converger vers le même état
- Ordre des opérations respecté (causalité)

**Availability (Disponibilité)**
- Chaque système accepte des writes même en cas de partition
- Pas de blocking sur synchronisation

**Partition Tolerance (Tolérance aux partitions)**
- Systèmes fonctionnent même si réseau coupé temporairement
- Réconciliation automatique après reconnexion

**Compromis inévitables (théorème CAP)**
```
Dans un système distribué, impossible d'avoir simultanément :
- Consistency forte
- Availability totale
- Partition tolerance

Choix typique pour sync bidirectionnelle :
→ AP (Available + Partition tolerant)
→ Eventually Consistent (cohérence à terme)
```

### Types de conflits

**1. Conflits d'écriture (Write-Write)**
```
SQL:     [user.name = "Alice"]  (T1)
MongoDB: [user.name = "Alicia"] (T1)

→ Conflit : Deux modifications concurrentes du même champ
```

**2. Conflits d'insertion (Insert-Insert)**
```
SQL:     INSERT user (id=123, name="Bob")    (T1)
MongoDB: INSERT user (id=123, name="Bobby")  (T1)

→ Conflit : Même clé, données différentes
```

**3. Conflits de suppression (Delete-Update)**
```
SQL:     DELETE user WHERE id=123  (T1)
MongoDB: UPDATE user SET age=30 WHERE id=123  (T1)

→ Conflit : Un système supprime, l'autre modifie
```

**4. Conflits de type (Type Mismatch)**
```
SQL:     [user.phone = "1234567890"] (STRING)
MongoDB: [user.phone = 1234567890]   (NUMBER)

→ Conflit : Incompatibilité de types
```

**5. Conflits structurels (Schema Evolution)**
```
SQL:     [user.address = "123 Main St"] (denormalized)
MongoDB: [user.address = {street: "123 Main St", city: "Paris"}] (object)

→ Conflit : Structure incompatible
```

---

## 🏗️ Patterns de Résolution de Conflits

### 1. Last-Write-Wins (LWW)

**Principe**
La dernière écriture (timestamp le plus récent) l'emporte.

**Implémentation**

```typescript
// lww-resolver.ts
interface VersionedRecord {
  id: string;
  data: any;
  version: number;
  timestamp: Date;
  source: 'sql' | 'mongodb';
}

class LWWResolver {
  resolve(
    sqlRecord: VersionedRecord,
    mongoRecord: VersionedRecord
  ): VersionedRecord {

    // Comparer timestamps
    if (sqlRecord.timestamp > mongoRecord.timestamp) {
      return {
        ...sqlRecord,
        resolvedBy: 'lww',
        winner: 'sql',
        timestamp: new Date()
      };
    } else if (mongoRecord.timestamp > sqlRecord.timestamp) {
      return {
        ...mongoRecord,
        resolvedBy: 'lww',
        winner: 'mongodb',
        timestamp: new Date()
      };
    } else {
      // Timestamps égaux → tiebreaker par source
      // (règle arbitraire : SQL gagne)
      return {
        ...sqlRecord,
        resolvedBy: 'lww-tiebreaker',
        winner: 'sql',
        timestamp: new Date()
      };
    }
  }
}
```

**Avantages**
- ✅ Simple à implémenter
- ✅ Déterministe (même résultat partout)
- ✅ Performance élevée

**Inconvénients**
- ❌ Perte possible de données (écriture écrasée)
- ❌ Pas de prise en compte de la sémantique métier
- ❌ Problème si clocks non synchronisées

**Quand utiliser ?**
- Données où la dernière valeur est la bonne (température, statut)
- Systèmes avec clock sync parfait (NTP)
- Taux de conflits faible

---

### 2. Version Vector

**Principe**
Chaque système maintient un vecteur de versions pour détecter causalité et conflits réels.

**Implémentation**

```typescript
// version-vector.ts
type VersionVector = {
  [systemId: string]: number;
};

interface VectoredRecord {
  id: string;
  data: any;
  versionVector: VersionVector;
}

class VersionVectorResolver {

  /**
   * Compare deux vecteurs de version
   *
   * Returns:
   * - 'before' : v1 happened before v2 (v1 < v2)
   * - 'after'  : v1 happened after v2 (v1 > v2)
   * - 'concurrent' : v1 and v2 are concurrent (conflict)
   * - 'equal' : v1 == v2 (no conflict)
   */
  compare(v1: VersionVector, v2: VersionVector): 'before' | 'after' | 'concurrent' | 'equal' {
    let v1Greater = false;
    let v2Greater = false;

    // Union de toutes les clés
    const allKeys = new Set([
      ...Object.keys(v1),
      ...Object.keys(v2)
    ]);

    for (const key of allKeys) {
      const val1 = v1[key] || 0;
      const val2 = v2[key] || 0;

      if (val1 > val2) v1Greater = true;
      if (val2 > val1) v2Greater = true;
    }

    if (!v1Greater && !v2Greater) return 'equal';
    if (v1Greater && !v2Greater) return 'after';
    if (!v1Greater && v2Greater) return 'before';
    return 'concurrent';  // Conflict réel !
  }

  resolve(
    sqlRecord: VectoredRecord,
    mongoRecord: VectoredRecord
  ): {
    action: 'use_sql' | 'use_mongo' | 'merge' | 'manual';
    resolved?: VectoredRecord;
  } {

    const comparison = this.compare(
      sqlRecord.versionVector,
      mongoRecord.versionVector
    );

    switch (comparison) {
      case 'equal':
        return { action: 'use_sql', resolved: sqlRecord };

      case 'after':
        // SQL is more recent
        return { action: 'use_sql', resolved: sqlRecord };

      case 'before':
        // MongoDB is more recent
        return { action: 'use_mongo', resolved: mongoRecord };

      case 'concurrent':
        // Vrai conflit → nécessite merge ou résolution manuelle
        return {
          action: 'merge',
          resolved: this.merge(sqlRecord, mongoRecord)
        };
    }
  }

  merge(
    sqlRecord: VectoredRecord,
    mongoRecord: VectoredRecord
  ): VectoredRecord {

    // Merge champ par champ avec stratégie customizable
    const mergedData: any = {};

    const allKeys = new Set([
      ...Object.keys(sqlRecord.data),
      ...Object.keys(mongoRecord.data)
    ]);

    for (const key of allKeys) {
      const sqlValue = sqlRecord.data[key];
      const mongoValue = mongoRecord.data[key];

      if (sqlValue === mongoValue) {
        mergedData[key] = sqlValue;
      } else if (sqlValue === undefined) {
        mergedData[key] = mongoValue;
      } else if (mongoValue === undefined) {
        mergedData[key] = sqlValue;
      } else {
        // Conflit sur ce champ → appliquer stratégie
        mergedData[key] = this.resolveFieldConflict(key, sqlValue, mongoValue);
      }
    }

    // Merge version vectors
    const mergedVector: VersionVector = {};
    const allSystems = new Set([
      ...Object.keys(sqlRecord.versionVector),
      ...Object.keys(mongoRecord.versionVector)
    ]);

    for (const system of allSystems) {
      mergedVector[system] = Math.max(
        sqlRecord.versionVector[system] || 0,
        mongoRecord.versionVector[system] || 0
      );
    }

    return {
      id: sqlRecord.id,
      data: mergedData,
      versionVector: mergedVector
    };
  }

  private resolveFieldConflict(field: string, sqlValue: any, mongoValue: any): any {
    // Stratégie par champ (configurable)
    const strategy = this.getFieldStrategy(field);

    switch (strategy) {
      case 'prefer_sql':
        return sqlValue;
      case 'prefer_mongo':
        return mongoValue;
      case 'prefer_non_null':
        return sqlValue !== null ? sqlValue : mongoValue;
      case 'merge_array':
        if (Array.isArray(sqlValue) && Array.isArray(mongoValue)) {
          return [...new Set([...sqlValue, ...mongoValue])];
        }
        return sqlValue;
      case 'merge_object':
        if (typeof sqlValue === 'object' && typeof mongoValue === 'object') {
          return { ...sqlValue, ...mongoValue };
        }
        return sqlValue;
      default:
        return sqlValue;
    }
  }

  private getFieldStrategy(field: string): string {
    // Configuration par champ
    const strategies: Record<string, string> = {
      'email': 'prefer_non_null',
      'tags': 'merge_array',
      'metadata': 'merge_object',
      'status': 'prefer_sql',
      'preferences': 'prefer_mongo'
    };

    return strategies[field] || 'prefer_sql';
  }
}

// Exemple d'utilisation
const resolver = new VersionVectorResolver();

const sqlRecord: VectoredRecord = {
  id: "user_123",
  data: {
    name: "Alice",
    email: "alice@example.com",
    age: 30
  },
  versionVector: {
    sql: 5,
    mongodb: 3
  }
};

const mongoRecord: VectoredRecord = {
  id: "user_123",
  data: {
    name: "Alice Smith",
    email: "alice@example.com",
    age: 30
  },
  versionVector: {
    sql: 4,
    mongodb: 6
  }
};

const result = resolver.resolve(sqlRecord, mongoRecord);
// result.action = 'concurrent'
// → Merge nécessaire
```

**Avantages**
- ✅ Détection précise des conflits réels
- ✅ Pas de perte de données causales
- ✅ Idéal pour systèmes distribués

**Inconvénients**
- ❌ Complexité implémentation
- ❌ Overhead stockage (vecteurs)
- ❌ Merge complexe si conflit

---

### 3. Operational Transformation (OT)

**Principe**
Transforme opérations concurrentes pour préserver intention utilisateur.

**Implémentation (texte collaboratif)**

```typescript
// operational-transformation.ts
interface Operation {
  type: 'insert' | 'delete' | 'retain';
  position: number;
  text?: string;
  length?: number;
}

class OTResolver {

  /**
   * Transform operation A against operation B
   * Used when both operations were applied to the same base state
   */
  transform(opA: Operation, opB: Operation): Operation {

    if (opA.type === 'insert' && opB.type === 'insert') {
      if (opA.position < opB.position) {
        return opA;  // A remains unchanged
      } else if (opA.position > opB.position) {
        return {
          ...opA,
          position: opA.position + (opB.text?.length || 0)
        };
      } else {
        // Same position → tiebreaker (arbitrary)
        return {
          ...opA,
          position: opA.position + (opB.text?.length || 0)
        };
      }
    }

    if (opA.type === 'insert' && opB.type === 'delete') {
      if (opA.position <= opB.position) {
        return opA;
      } else if (opA.position > opB.position + (opB.length || 0)) {
        return {
          ...opA,
          position: opA.position - (opB.length || 0)
        };
      } else {
        // Insert inside delete range
        return {
          ...opA,
          position: opB.position
        };
      }
    }

    if (opA.type === 'delete' && opB.type === 'insert') {
      if (opA.position < opB.position) {
        return opA;
      } else {
        return {
          ...opA,
          position: opA.position + (opB.text?.length || 0)
        };
      }
    }

    if (opA.type === 'delete' && opB.type === 'delete') {
      if (opA.position + (opA.length || 0) <= opB.position) {
        return opA;
      } else if (opA.position >= opB.position + (opB.length || 0)) {
        return {
          ...opA,
          position: opA.position - (opB.length || 0)
        };
      } else {
        // Overlapping deletes → adjust
        const newLength = Math.max(
          0,
          (opA.length || 0) - (opB.length || 0)
        );
        return {
          ...opA,
          position: Math.min(opA.position, opB.position),
          length: newLength
        };
      }
    }

    return opA;
  }

  /**
   * Apply transformation to document field
   */
  applyOT(
    baseValue: string,
    sqlOp: Operation,
    mongoOp: Operation
  ): string {

    // Transform operations
    const transformedSqlOp = this.transform(sqlOp, mongoOp);
    const transformedMongoOp = this.transform(mongoOp, sqlOp);

    // Apply both operations in sequence
    let result = baseValue;
    result = this.applyOperation(result, transformedSqlOp);
    result = this.applyOperation(result, transformedMongoOp);

    return result;
  }

  private applyOperation(text: string, op: Operation): string {
    switch (op.type) {
      case 'insert':
        return (
          text.slice(0, op.position) +
          (op.text || '') +
          text.slice(op.position)
        );

      case 'delete':
        return (
          text.slice(0, op.position) +
          text.slice(op.position + (op.length || 0))
        );

      default:
        return text;
    }
  }
}

// Exemple
const ot = new OTResolver();

const baseText = "Hello world";

const sqlOp: Operation = {
  type: 'insert',
  position: 6,
  text: 'beautiful '
};

const mongoOp: Operation = {
  type: 'delete',
  position: 0,
  length: 6
};

const result = ot.applyOT(baseText, sqlOp, mongoOp);
// Result: "beautiful world"
// (SQL insert + Mongo delete reconciled)
```

**Avantages**
- ✅ Préserve intention utilisateur
- ✅ Idéal pour édition collaborative (Google Docs style)
- ✅ Convergence garantie

**Inconvénients**
- ❌ Complexité algorithmique élevée
- ❌ Limité aux types séquentiels (texte)
- ❌ Performance sur grandes opérations

**Quand utiliser ?**
- Édition collaborative de texte
- Synchronisation documents riches
- Systèmes avec OT natif (MongoDB Realm Sync utilise OT)

---

### 4. CRDT (Conflict-free Replicated Data Types)

**Principe**
Types de données conçus mathématiquement pour converger automatiquement sans conflit.

**Implémentation G-Counter (Grow-only Counter)**

```typescript
// crdt-gcounter.ts
interface GCounter {
  id: string;
  counts: Map<string, number>;  // nodeId → count
}

class GCounterCRDT {

  increment(counter: GCounter, nodeId: string, amount: number = 1): GCounter {
    const newCounts = new Map(counter.counts);
    const current = newCounts.get(nodeId) || 0;
    newCounts.set(nodeId, current + amount);

    return {
      id: counter.id,
      counts: newCounts
    };
  }

  merge(counter1: GCounter, counter2: GCounter): GCounter {
    const mergedCounts = new Map<string, number>();

    // Union de tous les nodeIds
    const allNodes = new Set([
      ...counter1.counts.keys(),
      ...counter2.counts.keys()
    ]);

    for (const nodeId of allNodes) {
      const count1 = counter1.counts.get(nodeId) || 0;
      const count2 = counter2.counts.get(nodeId) || 0;

      // Prendre le max pour chaque node (monotonic increase)
      mergedCounts.set(nodeId, Math.max(count1, count2));
    }

    return {
      id: counter1.id,
      counts: mergedCounts
    };
  }

  value(counter: GCounter): number {
    let total = 0;
    for (const count of counter.counts.values()) {
      total += count;
    }
    return total;
  }
}

// PN-Counter (Positive-Negative Counter) - supports decrement
interface PNCounter {
  id: string;
  positive: Map<string, number>;
  negative: Map<string, number>;
}

class PNCounterCRDT {

  increment(counter: PNCounter, nodeId: string, amount: number = 1): PNCounter {
    const newPositive = new Map(counter.positive);
    const current = newPositive.get(nodeId) || 0;
    newPositive.set(nodeId, current + amount);

    return {
      ...counter,
      positive: newPositive
    };
  }

  decrement(counter: PNCounter, nodeId: string, amount: number = 1): PNCounter {
    const newNegative = new Map(counter.negative);
    const current = newNegative.get(nodeId) || 0;
    newNegative.set(nodeId, current + amount);

    return {
      ...counter,
      negative: newNegative
    };
  }

  merge(counter1: PNCounter, counter2: PNCounter): PNCounter {
    const mergedPositive = this.mergeMaps(counter1.positive, counter2.positive);
    const mergedNegative = this.mergeMaps(counter1.negative, counter2.negative);

    return {
      id: counter1.id,
      positive: mergedPositive,
      negative: mergedNegative
    };
  }

  private mergeMaps(
    map1: Map<string, number>,
    map2: Map<string, number>
  ): Map<string, number> {
    const merged = new Map<string, number>();
    const allKeys = new Set([...map1.keys(), ...map2.keys()]);

    for (const key of allKeys) {
      merged.set(key, Math.max(map1.get(key) || 0, map2.get(key) || 0));
    }

    return merged;
  }

  value(counter: PNCounter): number {
    let positive = 0;
    for (const count of counter.positive.values()) {
      positive += count;
    }

    let negative = 0;
    for (const count of counter.negative.values()) {
      negative += count;
    }

    return positive - negative;
  }
}

// LWW-Element-Set (Last-Write-Wins Set)
interface LWWElement<T> {
  value: T;
  timestamp: Date;
  tombstone: boolean;  // true if deleted
}

class LWWSetCRDT<T> {
  private elements: Map<string, LWWElement<T>>;

  constructor() {
    this.elements = new Map();
  }

  add(value: T, timestamp: Date = new Date()): void {
    const key = this.hash(value);
    const existing = this.elements.get(key);

    if (!existing || timestamp > existing.timestamp) {
      this.elements.set(key, {
        value,
        timestamp,
        tombstone: false
      });
    }
  }

  remove(value: T, timestamp: Date = new Date()): void {
    const key = this.hash(value);
    const existing = this.elements.get(key);

    if (!existing || timestamp > existing.timestamp) {
      this.elements.set(key, {
        value,
        timestamp,
        tombstone: true
      });
    }
  }

  merge(other: LWWSetCRDT<T>): LWWSetCRDT<T> {
    const merged = new LWWSetCRDT<T>();

    // Merge this elements
    for (const [key, element] of this.elements) {
      merged.elements.set(key, element);
    }

    // Merge other elements (keeping latest)
    for (const [key, otherElement] of other.elements) {
      const existing = merged.elements.get(key);

      if (!existing || otherElement.timestamp > existing.timestamp) {
        merged.elements.set(key, otherElement);
      }
    }

    return merged;
  }

  values(): T[] {
    const result: T[] = [];

    for (const element of this.elements.values()) {
      if (!element.tombstone) {
        result.push(element.value);
      }
    }

    return result;
  }

  private hash(value: T): string {
    return JSON.stringify(value);
  }
}
```

**Application pratique : Synchronisation tags**

```typescript
// Exemple : Tags utilisateur synchronisés SQL ↔ MongoDB
class UserTagsSyncService {
  private sqlTags: LWWSetCRDT<string>;
  private mongoTags: LWWSetCRDT<string>;

  async syncTags(userId: string): Promise<void> {
    // Récupérer tags SQL
    const sqlTagsData = await this.fetchSQLTags(userId);
    this.sqlTags = this.buildLWWSet(sqlTagsData);

    // Récupérer tags MongoDB
    const mongoTagsData = await this.fetchMongoTags(userId);
    this.mongoTags = this.buildLWWSet(mongoTagsData);

    // Merge automatique (pas de conflit !)
    const mergedTags = this.sqlTags.merge(this.mongoTags);

    // Écrire résultat dans les deux systèmes
    await Promise.all([
      this.writeSQLTags(userId, mergedTags.values()),
      this.writeMongoTags(userId, mergedTags.values())
    ]);
  }
}
```

**Avantages**
- ✅ Convergence automatique garantie
- ✅ Pas de résolution de conflit nécessaire
- ✅ Performance élevée

**Inconvénients**
- ❌ Types limités (compteurs, sets, registres)
- ❌ Overhead mémoire (métadonnées)
- ❌ Pas applicable à toutes les données

**Quand utiliser ?**
- Compteurs distribués (vues, likes)
- Sets collaboratifs (tags, labels)
- Données où merge mathématique possible

---

## 🔄 Architectures de Synchronisation Bidirectionnelle

### Architecture 1 : Event-Driven avec Kafka

**Principe**
Chaque système publie ses changements dans Kafka. Un orchestrateur réconcilie et propage.

**Architecture complète**

```
┌─────────────────────────────────────────────────────────────┐
│         Event-Driven Bidirectional Sync Architecture        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Application] ──writes──▶ [PostgreSQL]                     │
│                                  │                          │
│                                  │ CDC (Debezium)           │
│                                  ↓                          │
│                          [Kafka Topic: sql_changes]         │
│                                  ↓                          │
│                          ┌────────────────┐                 │
│                          │ Reconciliation │                 │
│                          │  Orchestrator  │                 │
│                          │                │                 │
│  [MongoDB] ──changes───▶ │  • Detect      │                 │
│      ↑                   │    conflicts   │                 │
│      │                   │  • Resolve     │                 │
│      │                   │  • Propagate   │                 │
│      │                   └───────┬────────┘                 │
│      │                           │                          │
│      │                           ↓                          │
│      │                   [Kafka Topic: mongo_changes]       │
│      │                           │                          │
│      └───────────────────────────┘                          │
│                    (bidirectional sync)                     │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Conflict Resolution Store (MongoDB)                   │ │
│  │  • Conflict log                                        │ │
│  │  • Resolution history                                  │ │
│  │  • Metrics & analytics                                 │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Implémentation Orchestrateur**

```typescript
// reconciliation-orchestrator.ts
import { Kafka, Consumer, Producer } from 'kafkajs';
import { MongoClient } from 'mongodb';
import { Client as PgClient } from 'pg';

interface ChangeEvent {
  source: 'sql' | 'mongodb';
  operation: 'insert' | 'update' | 'delete';
  table: string;
  key: any;
  before?: any;
  after?: any;
  timestamp: Date;
  version?: number;
}

interface ConflictResolution {
  conflictId: string;
  sqlEvent: ChangeEvent;
  mongoEvent: ChangeEvent;
  resolution: 'sql_wins' | 'mongo_wins' | 'merged';
  resolvedData: any;
  resolvedAt: Date;
  strategy: string;
}

class ReconciliationOrchestrator {
  private kafka: Kafka;
  private sqlConsumer: Consumer;
  private mongoConsumer: Consumer;
  private producer: Producer;
  private mongoClient: MongoClient;
  private pgClient: PgClient;

  // Buffer des événements récents pour détection conflits
  private eventBuffer: Map<string, ChangeEvent[]>;
  private conflictWindow: number = 5000;  // 5 secondes

  async start(): Promise<void> {
    // Setup Kafka
    this.kafka = new Kafka({
      brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092']
    });

    this.sqlConsumer = this.kafka.consumer({ groupId: 'sql-sync' });
    this.mongoConsumer = this.kafka.consumer({ groupId: 'mongo-sync' });
    this.producer = this.kafka.producer();

    await this.sqlConsumer.connect();
    await this.mongoConsumer.connect();
    await this.producer.connect();

    // Subscribe topics
    await this.sqlConsumer.subscribe({ topic: 'sql_changes' });
    await this.mongoConsumer.subscribe({ topic: 'mongo_changes' });

    // Setup databases
    this.mongoClient = new MongoClient(process.env.MONGODB_URI);
    await this.mongoClient.connect();

    this.pgClient = new PgClient(/* config */);
    await this.pgClient.connect();

    // Start consuming
    this.eventBuffer = new Map();

    await Promise.all([
      this.consumeSQLChanges(),
      this.consumeMongoChanges()
    ]);
  }

  private async consumeSQLChanges(): Promise<void> {
    await this.sqlConsumer.run({
      eachMessage: async ({ message }) => {
        const event: ChangeEvent = JSON.parse(message.value.toString());
        event.source = 'sql';

        await this.processEvent(event);
      }
    });
  }

  private async consumeMongoChanges(): Promise<void> {
    await this.mongoConsumer.run({
      eachMessage: async ({ message }) => {
        const event: ChangeEvent = JSON.parse(message.value.toString());
        event.source = 'mongodb';

        await this.processEvent(event);
      }
    });
  }

  private async processEvent(event: ChangeEvent): Promise<void> {
    const eventKey = this.getEventKey(event);

    // Ajouter à buffer
    if (!this.eventBuffer.has(eventKey)) {
      this.eventBuffer.set(eventKey, []);
    }
    this.eventBuffer.get(eventKey).push(event);

    // Nettoyer buffer ancien
    this.cleanBuffer(eventKey);

    // Détecter conflits
    const conflictingEvents = this.detectConflicts(eventKey);

    if (conflictingEvents.length > 1) {
      // Conflit détecté !
      await this.resolveConflict(conflictingEvents);
    } else {
      // Pas de conflit → propager
      await this.propagateEvent(event);
    }
  }

  private detectConflicts(eventKey: string): ChangeEvent[] {
    const events = this.eventBuffer.get(eventKey) || [];

    // Grouper par source
    const sqlEvents = events.filter(e => e.source === 'sql');
    const mongoEvents = events.filter(e => e.source === 'mongodb');

    // Conflit si événements des deux sources dans la fenêtre
    if (sqlEvents.length > 0 && mongoEvents.length > 0) {
      const latestSql = sqlEvents[sqlEvents.length - 1];
      const latestMongo = mongoEvents[mongoEvents.length - 1];

      // Vérifier si vraiment concurrent (pas causal)
      const timeDiff = Math.abs(
        latestSql.timestamp.getTime() - latestMongo.timestamp.getTime()
      );

      if (timeDiff < this.conflictWindow) {
        return [latestSql, latestMongo];
      }
    }

    return events.slice(-1);  // Pas de conflit, retourner le dernier
  }

  private async resolveConflict(events: ChangeEvent[]): Promise<void> {
    const [sqlEvent, mongoEvent] = events;

    console.log('Conflict detected:', {
      key: this.getEventKey(sqlEvent),
      sqlTimestamp: sqlEvent.timestamp,
      mongoTimestamp: mongoEvent.timestamp
    });

    // Appliquer stratégie de résolution
    const resolution = await this.applyResolutionStrategy(sqlEvent, mongoEvent);

    // Logger conflit
    await this.logConflict(resolution);

    // Propager résolution
    await this.propagateResolution(resolution);

    // Métriques
    this.recordConflictMetric(resolution);
  }

  private async applyResolutionStrategy(
    sqlEvent: ChangeEvent,
    mongoEvent: ChangeEvent
  ): Promise<ConflictResolution> {

    const strategy = this.getStrategyForTable(sqlEvent.table);

    switch (strategy) {
      case 'lww':
        return this.resolveLWW(sqlEvent, mongoEvent);

      case 'version_vector':
        return this.resolveVersionVector(sqlEvent, mongoEvent);

      case 'merge':
        return this.resolveMerge(sqlEvent, mongoEvent);

      case 'manual':
        return this.queueForManualResolution(sqlEvent, mongoEvent);

      default:
        return this.resolveLWW(sqlEvent, mongoEvent);
    }
  }

  private resolveLWW(
    sqlEvent: ChangeEvent,
    mongoEvent: ChangeEvent
  ): ConflictResolution {

    const sqlWins = sqlEvent.timestamp > mongoEvent.timestamp;

    return {
      conflictId: this.generateConflictId(),
      sqlEvent,
      mongoEvent,
      resolution: sqlWins ? 'sql_wins' : 'mongo_wins',
      resolvedData: sqlWins ? sqlEvent.after : mongoEvent.after,
      resolvedAt: new Date(),
      strategy: 'lww'
    };
  }

  private async resolveMerge(
    sqlEvent: ChangeEvent,
    mongoEvent: ChangeEvent
  ): Promise<ConflictResolution> {

    // Merge champ par champ
    const merged = {
      ...sqlEvent.after,
      ...mongoEvent.after
    };

    // Appliquer règles métier spécifiques
    merged.name = mongoEvent.after?.name || sqlEvent.after?.name;
    merged.email = sqlEvent.after?.email || mongoEvent.after?.email;
    merged.updated_at = new Date();

    return {
      conflictId: this.generateConflictId(),
      sqlEvent,
      mongoEvent,
      resolution: 'merged',
      resolvedData: merged,
      resolvedAt: new Date(),
      strategy: 'merge'
    };
  }

  private async propagateResolution(resolution: ConflictResolution): Promise<void> {
    // Écrire dans SQL
    if (resolution.resolution !== 'sql_wins') {
      await this.writeSQLWithRetry(
        resolution.sqlEvent.table,
        resolution.sqlEvent.key,
        resolution.resolvedData
      );
    }

    // Écrire dans MongoDB
    if (resolution.resolution !== 'mongo_wins') {
      await this.writeMongoWithRetry(
        resolution.mongoEvent.table,
        resolution.mongoEvent.key,
        resolution.resolvedData
      );
    }
  }

  private async propagateEvent(event: ChangeEvent): Promise<void> {
    // Propager vers l'autre système
    if (event.source === 'sql') {
      await this.writeMongoWithRetry(event.table, event.key, event.after);
    } else {
      await this.writeSQLWithRetry(event.table, event.key, event.after);
    }
  }

  private async logConflict(resolution: ConflictResolution): Promise<void> {
    await this.mongoClient
      .db('sync_metadata')
      .collection('conflicts')
      .insertOne({
        ...resolution,
        created_at: new Date()
      });
  }

  private getEventKey(event: ChangeEvent): string {
    return `${event.table}:${JSON.stringify(event.key)}`;
  }

  private cleanBuffer(eventKey: string): void {
    const events = this.eventBuffer.get(eventKey) || [];
    const now = Date.now();

    const filtered = events.filter(
      e => now - e.timestamp.getTime() < this.conflictWindow * 2
    );

    this.eventBuffer.set(eventKey, filtered);
  }

  private getStrategyForTable(table: string): string {
    const strategies: Record<string, string> = {
      'users': 'merge',
      'orders': 'lww',
      'products': 'version_vector',
      'settings': 'manual'
    };

    return strategies[table] || 'lww';
  }

  private generateConflictId(): string {
    return `conflict_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private async writeSQLWithRetry(table: string, key: any, data: any): Promise<void> {
    // Implémentation avec retry logic
  }

  private async writeMongoWithRetry(collection: string, key: any, data: any): Promise<void> {
    // Implémentation avec retry logic
  }
}
```

**Monitoring des conflits**

```typescript
// conflict-dashboard.ts
interface ConflictMetrics {
  totalConflicts: number;
  conflictsLast24h: number;
  conflictsByTable: Record<string, number>;
  resolutionStrategies: Record<string, number>;
  avgResolutionTime: number;
  pendingManualResolutions: number;
}

class ConflictMonitor {
  async getMetrics(timeRange: string = '24h'): Promise<ConflictMetrics> {
    const db = this.mongoClient.db('sync_metadata');

    const startTime = this.getStartTime(timeRange);

    const conflicts = await db
      .collection('conflicts')
      .find({ created_at: { $gte: startTime } })
      .toArray();

    const totalConflicts = conflicts.length;

    // Group by table
    const conflictsByTable: Record<string, number> = {};
    for (const conflict of conflicts) {
      const table = conflict.sqlEvent.table;
      conflictsByTable[table] = (conflictsByTable[table] || 0) + 1;
    }

    // Group by resolution strategy
    const resolutionStrategies: Record<string, number> = {};
    for (const conflict of conflicts) {
      const strategy = conflict.strategy;
      resolutionStrategies[strategy] = (resolutionStrategies[strategy] || 0) + 1;
    }

    // Average resolution time
    const resolutionTimes = conflicts.map(c =>
      c.resolvedAt.getTime() - c.created_at.getTime()
    );
    const avgResolutionTime = resolutionTimes.reduce((a, b) => a + b, 0) / resolutionTimes.length;

    // Pending manual resolutions
    const pendingManual = await db
      .collection('conflicts')
      .countDocuments({
        strategy: 'manual',
        resolved: false
      });

    return {
      totalConflicts,
      conflictsLast24h: totalConflicts,
      conflictsByTable,
      resolutionStrategies,
      avgResolutionTime,
      pendingManualResolutions: pendingManual
    };
  }
}
```

---

## 📊 Scénario Réel : E-commerce Global (Sync Bidirectionnelle 9 mois)

**Contexte**
- E-commerce international, 50M produits
- Système legacy : Oracle 19c (5 To)
- Cible : MongoDB Atlas (multi-région)
- Contrainte : Zero downtime, sync bidirectionnelle pendant migration (6 mois)
- Équipe : 12 devs, 3 DBAs, 2 architectes

### Architecture implémentée

```
┌─────────────────────────────────────────────────────────────┐
│              Production Architecture                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Web App] ──┬──▶ [API Gateway (Kong)]                      │
│              │         │                                    │
│              │         ├──▶ [Products Service]              │
│              │         │      ↓                             │
│              │         │   [Oracle] (writes)                │
│              │         │      ↓                             │
│              │         │   [Debezium CDC]                   │
│              │         │      ↓                             │
│              │         │   [Kafka Cluster]                  │
│              │         │      ↓                             │
│              │         │   [Reconciliation Orchestrator]    │
│              │         │      ↓                             │
│              │         │   [MongoDB Atlas]                  │
│              │         │      ↑                             │
│              │         │   [Change Streams]                 │
│              │         │      ↑                             │
│              │         └────────┘                           │
│              │                                              │
│              └──▶ [Search Service]                          │
│                         ↓                                   │
│                      [MongoDB Atlas] (reads)                │
└─────────────────────────────────────────────────────────────┘
```

### Timeline

**Mois 1-2 : Setup Infrastructure**
- Kafka cluster (9 brokers, 3 AZs)
- Debezium Oracle connector
- MongoDB Atlas M80 clusters (3 régions)
- Reconciliation orchestrator (custom)

**Mois 3-4 : Migration Initiale + CDC**
- Snapshot 50M produits → MongoDB (72h)
- Activation CDC Oracle → Kafka → MongoDB
- Validation : 100% données synchronisées
- Lag CDC : < 500ms P95

**Mois 5-6 : Phase Dual-Read**
- API Gateway route 10% reads → MongoDB
- Comparaison automatisée (1M queries/jour)
- Détection divergences : 0.02% (fixées)
- Performance MongoDB : +45% vs Oracle

**Mois 7 : Bascule Reads 100% MongoDB**
- Rollout progressif : 10% → 50% → 100%
- Oracle devient read-only (fallback uniquement)
- Monitoring intensif

**Mois 8 : Phase Dual-Write Bidirectionnelle**
- Writes vont vers Oracle ET MongoDB
- Sync bidirectionnelle active
- Conflits détectés : 234 en 1 mois
  - LWW : 180 (77%)
  - Merge : 45 (19%)
  - Manual : 9 (4%)

**Mois 9 : MongoDB devient Primary**
- Arrêt dual-write
- MongoDB = source de vérité
- Oracle = archive (6 mois retention)

### Résultats

**Performance**
- Latence lecture : 180ms → 45ms (P95)
- Throughput : 8K → 25K req/sec
- Query complexes : 5s → 800ms

**Fiabilité**
- Uptime : 99.98% (objectif 99.95%)
- Zero data loss
- 234 conflits automatiquement résolus
- 9 résolutions manuelles (< 24h chacune)

**Business**
- Recherche produits 5x plus rapide
- Support multi-région natif
- Coûts infra : -35%

### Challenges et Solutions

**Challenge 1 : CDC lag spikes Oracle**
- **Symptôme** : Lag monte à 30 secondes lors des imports bulk (nuit)
- **Cause** : Bulk updates génèrent des millions d'events
- **Solution** :
  - Batch processing avec throttling
  - Augmentation partitions Kafka (12 → 24)
  - Buffer size optimisé

**Challenge 2 : Conflits sur prix produits**
- **Symptôme** : 120 conflits/jour sur champ `price`
- **Cause** : Mise à jour manuelle Oracle + automatique MongoDB (algorithmes pricing)
- **Solution** :
  - Oracle = source vérité pour prix manuels
  - MongoDB = source pour prix calculés
  - Flag `price_source` pour arbitrage

**Challenge 3 : Performance MongoDB dégradée (1 query)**
- **Symptôme** : Query "products by category" 10x plus lente MongoDB
- **Cause** : $lookup sur 3 collections (over-normalisé)
- **Solution** : Refactoring modèle avec embedded categories

**Challenge 4 : Divergence images produits**
- **Symptôme** : 0.5% divergence URLs images
- **Cause** : Race condition dual-write + CDN cache
- **Solution** :
  - Versioning URLs images
  - Cache invalidation synchronisée
  - Reconciliation asynchrone

---

## 🎯 Bonnes Pratiques

### 1. Choisir la bonne stratégie de résolution

```typescript
// Décision tree pour stratégie de résolution
function chooseResolutionStrategy(
  dataType: string,
  conflictRate: number,
  criticalityLevel: 'low' | 'medium' | 'high'
): string {

  // Données critiques → Manual review
  if (criticalityLevel === 'high' && conflictRate > 0.001) {
    return 'manual';
  }

  // Texte collaboratif → OT/CRDT
  if (dataType === 'text' || dataType === 'document') {
    return 'ot';
  }

  // Compteurs, sets → CRDT
  if (dataType === 'counter' || dataType === 'set') {
    return 'crdt';
  }

  // Données avec sémantique métier forte → Merge
  if (dataType === 'user_profile' || dataType === 'settings') {
    return 'merge';
  }

  // Défaut → LWW
  return 'lww';
}
```

### 2. Monitoring exhaustif

```typescript
// Métriques essentielles à surveiller
interface SyncMetrics {
  // Performance
  syncLatency: {
    p50: number;
    p95: number;
    p99: number;
  };
  throughput: number;

  // Qualité
  conflictRate: number;
  divergenceRate: number;
  autoResolutionRate: number;
  manualResolutionPending: number;

  // Fiabilité
  errorRate: number;
  retryRate: number;
  dlqMessages: number;

  // Lag
  cdcLag: number;
  kafkaLag: number;
  replicationLag: number;
}

// Alertes critiques
const ALERT_THRESHOLDS = {
  conflictRate: 0.01,       // > 1%
  divergenceRate: 0.001,    // > 0.1%
  syncLatency: 5000,        // > 5s
  cdcLag: 60000,           // > 1 minute
  errorRate: 0.05          // > 5%
};
```

### 3. Testing de synchronisation

```typescript
// Test de synchronisation bidirectionnelle
describe('Bidirectional Sync', () => {

  test('should handle concurrent updates', async () => {
    const userId = 'test_user_123';

    // Write concurrently to both systems
    await Promise.all([
      sqlClient.query(
        'UPDATE users SET name = $1 WHERE id = $2',
        ['Alice SQL', userId]
      ),
      mongoDb.collection('users').updateOne(
        { _id: userId },
        { $set: { name: 'Alice Mongo' } }
      )
    ]);

    // Wait for sync
    await sleep(5000);

    // Both should converge to same value
    const sqlUser = await sqlClient.query(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    const mongoUser = await mongoDb.collection('users').findOne({
      _id: userId
    });

    expect(sqlUser.name).toBe(mongoUser.name);
  });

  test('should handle delete-update conflict', async () => {
    const userId = 'test_user_456';

    // Delete in SQL, Update in MongoDB
    await Promise.all([
      sqlClient.query('DELETE FROM users WHERE id = $1', [userId]),
      mongoDb.collection('users').updateOne(
        { _id: userId },
        { $set: { age: 30 } }
      )
    ]);

    await sleep(5000);

    // Verify resolution (delete should win)
    const sqlExists = await sqlClient.query(
      'SELECT COUNT(*) FROM users WHERE id = $1',
      [userId]
    );
    const mongoExists = await mongoDb.collection('users').countDocuments({
      _id: userId
    });

    expect(sqlExists.count).toBe(0);
    expect(mongoExists).toBe(0);
  });
});
```

### 4. Plan de rollback

```yaml
# Rollback plan par niveau de criticité

level_1_minor_divergence:
  trigger: divergence_rate < 0.1%
  action:
    - Continue sync
    - Monitor closely
    - Log for analysis

level_2_moderate_divergence:
  trigger: divergence_rate 0.1% - 1%
  action:
    - Alert on-call
    - Increase reconciliation frequency
    - Manual review of divergent records

level_3_major_divergence:
  trigger: divergence_rate > 1%
  action:
    - Pause sync
    - Full reconciliation
    - Root cause analysis
    - Resume after fix

level_4_critical_failure:
  trigger: sync completely broken
  action:
    - Immediate rollback to single source
    - Activate fallback (SQL or MongoDB)
    - Emergency fix
    - Phased restart
```

---

## 📚 Checklist Synchronisation Bidirectionnelle

**Avant de commencer**
- [ ] Architecture de résolution de conflits définie
- [ ] Stratégies par type de données documentées
- [ ] Infrastructure Kafka provisionnée
- [ ] Monitoring et alerting configurés
- [ ] Plan de rollback testé
- [ ] Équipe formée sur résolution conflits

**Pendant la synchronisation**
- [ ] Lag CDC < 1 seconde (P95)
- [ ] Taux de conflits < 1%
- [ ] Taux de divergence < 0.1%
- [ ] Taux de résolution auto > 95%
- [ ] Résolutions manuelles < 24h
- [ ] Monitoring temps réel actif

**Validation continue**
- [ ] Comparaison automatisée quotidienne
- [ ] Tests fonctionnels end-to-end
- [ ] Audit des conflits résolus
- [ ] Performance acceptable (latence, throughput)
- [ ] Aucune perte de données détectée

**Post-synchronisation**
- [ ] Analyse des conflits (patterns, causes)
- [ ] Documentation des cas edge
- [ ] Optimisation stratégies résolution
- [ ] Formation équipe sur leçons apprises
- [ ] Post-mortem et amélioration continue

---

## 🎓 Conclusion

La synchronisation bidirectionnelle est **complexe** mais **nécessaire** pour les migrations zero-downtime de systèmes critiques. Les points clés :

**Choix de la stratégie**
- LWW : Simple, perte de données possible
- Version Vector : Détection précise, merge complexe
- OT : Idéal pour texte collaboratif
- CRDT : Convergence automatique, types limités

**Architecture**
- Event-driven avec Kafka recommandé
- Orchestrateur centralisé pour réconciliation
- Monitoring exhaustif essentiel

**Gestion des conflits**
- 95%+ résolution automatique possible
- Plan pour résolutions manuelles nécessaire
- Tests exhaustifs critiques

**Réalisme**
- Prévoir 6-12 mois pour production
- Budget conséquent (infrastructure double)
- Expertise distribuée requise

La synchronisation bidirectionnelle n'est pas une solution long-terme mais une **phase transitoire** vers MongoDB. Objectif : Converger vers source unique dès que possible.

---

**Prochaine section** : 19.6 MongoDB Connector for BI - Intégration de MongoDB avec outils BI traditionnels.

⏭️ [MongoDB Connector for BI](/19-migration-integration/06-mongodb-connector-bi.md)
