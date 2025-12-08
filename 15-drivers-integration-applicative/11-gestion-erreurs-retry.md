🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.11 Gestion des erreurs et retry

## Introduction

La **gestion des erreurs** et les **stratégies de retry** sont essentielles pour construire des applications MongoDB robustes et résilientes. Les erreurs peuvent survenir pour diverses raisons : problèmes réseau, élections de replica set, timeouts, contraintes de données. Savoir identifier, classifier et gérer ces erreurs de manière appropriée garantit la stabilité et la disponibilité de vos applications.

## Types d'erreurs MongoDB

### Classification des erreurs

```
┌────────────────────────────────────────────────┐
│           Erreurs MongoDB                      │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │   1. Erreurs Transitoires                │  │
│  │   (Retry recommandé)                     │  │
│  │   - Network errors                       │  │
│  │   - Replica set elections                │  │
│  │   - Primary stepped down                 │  │
│  │   - Connection timeouts                  │  │
│  │   - Socket errors                        │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │   2. Erreurs Permanentes                 │  │
│  │   (Ne pas retry)                         │  │
│  │   - Authentication failed                │  │
│  │   - Duplicate key (E11000)               │  │
│  │   - Validation errors                    │  │
│  │   - Syntax errors                        │  │
│  │   - Authorization denied                 │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │   3. Erreurs de Transaction              │  │
│  │   (Retry avec stratégie spéciale)        │  │
│  │   - TransientTransactionError            │  │
│  │   - UnknownTransactionCommitResult       │  │
│  │   - WriteConflict                        │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │   4. Erreurs de Timeout                  │  │
│  │   (Retry selon contexte)                 │  │
│  │   - MaxTimeMSExpired                     │  │
│  │   - ConnectionTimeout                    │  │
│  │   - SocketTimeout                        │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Codes d'erreur principaux

```javascript
const ErrorCodes = {
  // Erreurs réseau (transitoires)
  NETWORK_TIMEOUT: 'NetworkTimeout',
  SOCKET_EXCEPTION: 'SocketException',
  HOST_UNREACHABLE: 'HostUnreachable',
  HOST_NOT_FOUND: 'HostNotFound',

  // Erreurs de replica set (transitoires)
  NOT_PRIMARY: 'NotPrimary',
  NOT_PRIMARY_NO_SECONDARY_OK: 'NotPrimaryNoSecondaryOk',
  NOT_PRIMARY_OR_SECONDARY: 'NotPrimaryOrSecondary',
  PRIMARY_STEPPED_DOWN: 'PrimarySteppedDown',
  INTERRUPTED_AT_SHUTDOWN: 'InterruptedAtShutdown',
  INTERRUPTED_DUE_TO_REPL_STATE_CHANGE: 'InterruptedDueToReplStateChange',

  // Erreurs d'authentification (permanentes)
  AUTHENTICATION_FAILED: 13,
  UNAUTHORIZED: 13,

  // Erreurs de données (permanentes)
  DUPLICATE_KEY: 11000,
  DOCUMENT_VALIDATION_FAILURE: 121,

  // Erreurs de transaction (retry spécial)
  TRANSIENT_TRANSACTION_ERROR: 'TransientTransactionError',
  UNKNOWN_TRANSACTION_COMMIT_RESULT: 'UnknownTransactionCommitResult',
  WRITE_CONFLICT: 112,

  // Erreurs de timeout
  MAX_TIME_MS_EXPIRED: 50,
  EXECUTION_TIMEOUT: 'ExceededTimeLimit'
};
```

## Stratégies de retry

### Retry automatique (Driver)

Les drivers MongoDB modernes incluent un retry automatique pour certaines opérations :

```javascript
// Node.js - Configuration du retry automatique
const client = new MongoClient(uri, {
  retryWrites: true,    // Retry automatique des writes
  retryReads: true      // Retry automatique des reads (MongoDB 4.4+)
});

// Le driver retry automatiquement pour :
// - Erreurs réseau transitoires
// - Elections de replica set
// - Primary stepped down
//
// Comportement :
// 1. Première tentative échoue
// 2. Driver attend un court moment
// 3. Driver retry une fois automatiquement
// 4. Si échec, propage l'erreur à l'application
```

### Retry manuel (Application)

Pour un contrôle plus fin, implémenter un retry manuel :

```javascript
// Pattern de base du retry
async function retryOperation(operation, maxRetries = 3, delayMs = 100) {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;

      // Vérifier si l'erreur est retryable
      if (!isRetryableError(error)) {
        throw error;
      }

      // Dernier essai ?
      if (attempt === maxRetries) {
        throw error;
      }

      // Attendre avant retry (avec backoff)
      const delay = delayMs * Math.pow(2, attempt - 1);
      await sleep(delay);

      console.log(`Retry attempt ${attempt} after ${delay}ms`);
    }
  }

  throw lastError;
}

function isRetryableError(error) {
  // Erreurs réseau
  if (error.name === 'MongoNetworkError') return true;
  if (error.name === 'MongoTimeoutError') return true;

  // Erreurs de replica set
  if (error.code === 11600) return true;  // InterruptedAtShutdown
  if (error.code === 11602) return true;  // InterruptedDueToReplStateChange
  if (error.code === 13436) return true;  // NotPrimaryOrSecondary
  if (error.code === 189) return true;    // PrimarySteppedDown

  // Transaction errors
  if (error.hasErrorLabel('TransientTransactionError')) return true;

  return false;
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

## Implémentations par langage

### Node.js / JavaScript

```javascript
// Classe complète de gestion des erreurs
class MongoErrorHandler {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.baseDelayMs = options.baseDelayMs || 100;
    this.maxDelayMs = options.maxDelayMs || 5000;
    this.logger = options.logger || console;
  }

  // Classifier une erreur
  classifyError(error) {
    // Erreurs transitoires
    if (this.isTransientError(error)) {
      return 'TRANSIENT';
    }

    // Erreurs de transaction
    if (this.isTransactionError(error)) {
      return 'TRANSACTION';
    }

    // Erreurs permanentes
    if (this.isPermanentError(error)) {
      return 'PERMANENT';
    }

    // Erreurs de timeout
    if (this.isTimeoutError(error)) {
      return 'TIMEOUT';
    }

    return 'UNKNOWN';
  }

  isTransientError(error) {
    // Erreurs réseau
    if (error.name === 'MongoNetworkError') return true;
    if (error.name === 'MongoServerSelectionError') return true;

    // Codes d'erreur transitoires
    const transientCodes = [11600, 11602, 13436, 189, 91];
    if (transientCodes.includes(error.code)) return true;

    // Messages spécifiques
    const errorMsg = error.message?.toLowerCase() || '';
    if (errorMsg.includes('socket') ||
        errorMsg.includes('network') ||
        errorMsg.includes('connection')) {
      return true;
    }

    return false;
  }

  isTransactionError(error) {
    return error.hasErrorLabel?.('TransientTransactionError') ||
           error.hasErrorLabel?.('UnknownTransactionCommitResult');
  }

  isPermanentError(error) {
    // Duplicate key
    if (error.code === 11000) return true;

    // Authentication
    if (error.code === 13 || error.code === 18) return true;

    // Validation
    if (error.code === 121) return true;

    // Syntax errors
    if (error.name === 'MongoParseError') return true;

    return false;
  }

  isTimeoutError(error) {
    if (error.name === 'MongoTimeoutError') return true;
    if (error.code === 50) return true; // MaxTimeMSExpired
    return false;
  }

  // Calculer le délai avec exponential backoff + jitter
  calculateDelay(attempt) {
    const exponentialDelay = this.baseDelayMs * Math.pow(2, attempt - 1);
    const jitter = Math.random() * 0.3 * exponentialDelay; // ±30% jitter
    const delay = Math.min(exponentialDelay + jitter, this.maxDelayMs);
    return Math.floor(delay);
  }

  // Retry avec stratégie adaptative
  async retry(operation, context = {}) {
    let lastError;

    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;

        const errorType = this.classifyError(error);

        this.logger.warn(`Operation failed (attempt ${attempt}/${this.maxRetries})`, {
          errorType,
          errorMessage: error.message,
          errorCode: error.code,
          context
        });

        // Erreur permanente : ne pas retry
        if (errorType === 'PERMANENT') {
          this.logger.error('Permanent error, not retrying', { error });
          throw error;
        }

        // Dernier essai
        if (attempt === this.maxRetries) {
          this.logger.error(`Max retries (${this.maxRetries}) reached`, { error });
          throw error;
        }

        // Calculer et attendre
        const delay = this.calculateDelay(attempt);
        this.logger.info(`Retrying in ${delay}ms...`);
        await this.sleep(delay);
      }
    }

    throw lastError;
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Utilisation
const errorHandler = new MongoErrorHandler({
  maxRetries: 3,
  baseDelayMs: 100,
  maxDelayMs: 5000
});

// Exemple : Insertion avec retry
async function insertUserWithRetry(userData) {
  return await errorHandler.retry(async () => {
    const collection = db.collection('users');
    return await collection.insertOne(userData);
  }, { operation: 'insertUser', userId: userData._id });
}

// Exemple : Transaction avec retry
async function transferWithRetry(fromId, toId, amount) {
  return await errorHandler.retry(async () => {
    const session = client.startSession();
    try {
      await session.withTransaction(async () => {
        await accountsCollection.updateOne(
          { _id: fromId },
          { $inc: { balance: -amount } },
          { session }
        );

        await accountsCollection.updateOne(
          { _id: toId },
          { $inc: { balance: amount } },
          { session }
        );
      });
    } finally {
      await session.endSession();
    }
  }, { operation: 'transfer', fromId, toId, amount });
}
```

### Python

```python
import time
import random
import logging
from pymongo.errors import (
    NetworkTimeout,
    ServerSelectionTimeoutError,
    ConnectionFailure,
    OperationFailure,
    DuplicateKeyError,
    PyMongoError
)

class MongoErrorHandler:
    def __init__(self, max_retries=3, base_delay_ms=100, max_delay_ms=5000):
        self.max_retries = max_retries
        self.base_delay_ms = base_delay_ms
        self.max_delay_ms = max_delay_ms
        self.logger = logging.getLogger(__name__)

    def classify_error(self, error):
        """Classifier une erreur"""
        if self.is_transient_error(error):
            return 'TRANSIENT'

        if self.is_transaction_error(error):
            return 'TRANSACTION'

        if self.is_permanent_error(error):
            return 'PERMANENT'

        if self.is_timeout_error(error):
            return 'TIMEOUT'

        return 'UNKNOWN'

    def is_transient_error(self, error):
        """Vérifier si erreur transitoire"""
        # Erreurs réseau
        if isinstance(error, (NetworkTimeout, ServerSelectionTimeoutError, ConnectionFailure)):
            return True

        # Codes transitoires
        if isinstance(error, OperationFailure):
            transient_codes = [11600, 11602, 13436, 189, 91]
            if error.code in transient_codes:
                return True

        return False

    def is_transaction_error(self, error):
        """Vérifier si erreur de transaction"""
        if isinstance(error, OperationFailure):
            return (error.has_error_label('TransientTransactionError') or
                    error.has_error_label('UnknownTransactionCommitResult'))
        return False

    def is_permanent_error(self, error):
        """Vérifier si erreur permanente"""
        # Duplicate key
        if isinstance(error, DuplicateKeyError):
            return True

        # Authentication/Authorization
        if isinstance(error, OperationFailure):
            if error.code in [13, 18, 121]:
                return True

        return False

    def is_timeout_error(self, error):
        """Vérifier si erreur de timeout"""
        if isinstance(error, NetworkTimeout):
            return True

        if isinstance(error, OperationFailure):
            if error.code == 50:  # MaxTimeMSExpired
                return True

        return False

    def calculate_delay(self, attempt):
        """Calculer délai avec exponential backoff + jitter"""
        exponential_delay = self.base_delay_ms * (2 ** (attempt - 1))
        jitter = random.uniform(-0.3, 0.3) * exponential_delay
        delay = min(exponential_delay + jitter, self.max_delay_ms)
        return delay / 1000  # Convertir en secondes

    def retry(self, operation, context=None):
        """Retry avec stratégie adaptative"""
        last_error = None
        context = context or {}

        for attempt in range(1, self.max_retries + 1):
            try:
                return operation()
            except PyMongoError as error:
                last_error = error
                error_type = self.classify_error(error)

                self.logger.warning(
                    f"Operation failed (attempt {attempt}/{self.max_retries})",
                    extra={
                        'error_type': error_type,
                        'error_message': str(error),
                        'context': context
                    }
                )

                # Erreur permanente
                if error_type == 'PERMANENT':
                    self.logger.error('Permanent error, not retrying')
                    raise

                # Dernier essai
                if attempt == self.max_retries:
                    self.logger.error(f'Max retries ({self.max_retries}) reached')
                    raise

                # Attendre avant retry
                delay = self.calculate_delay(attempt)
                self.logger.info(f'Retrying in {delay:.2f}s...')
                time.sleep(delay)

        raise last_error

# Utilisation
error_handler = MongoErrorHandler(max_retries=3, base_delay_ms=100)

def insert_user_with_retry(user_data):
    def operation():
        collection = db['users']
        return collection.insert_one(user_data)

    return error_handler.retry(
        operation,
        context={'operation': 'insertUser', 'user_id': user_data.get('_id')}
    )

def transfer_with_retry(from_id, to_id, amount):
    def operation():
        with client.start_session() as session:
            with session.start_transaction():
                accounts.update_one(
                    {'_id': from_id},
                    {'$inc': {'balance': -amount}},
                    session=session
                )

                accounts.update_one(
                    {'_id': to_id},
                    {'$inc': {'balance': amount}},
                    session=session
                )

    return error_handler.retry(
        operation,
        context={'operation': 'transfer', 'from': from_id, 'to': to_id}
    )
```

### Java

```java
import com.mongodb.MongoException;
import com.mongodb.MongoWriteException;
import com.mongodb.MongoTimeoutException;
import com.mongodb.MongoSocketException;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;
import java.util.function.Supplier;
import java.util.logging.Logger;

public class MongoErrorHandler {
    private static final Logger logger = Logger.getLogger(MongoErrorHandler.class.getName());

    private final int maxRetries;
    private final long baseDelayMs;
    private final long maxDelayMs;

    private static final Set<Integer> TRANSIENT_ERROR_CODES = new HashSet<>(
        Arrays.asList(11600, 11602, 13436, 189, 91)
    );

    public MongoErrorHandler(int maxRetries, long baseDelayMs, long maxDelayMs) {
        this.maxRetries = maxRetries;
        this.baseDelayMs = baseDelayMs;
        this.maxDelayMs = maxDelayMs;
    }

    public enum ErrorType {
        TRANSIENT, TRANSACTION, PERMANENT, TIMEOUT, UNKNOWN
    }

    public ErrorType classifyError(Exception error) {
        if (isTransientError(error)) {
            return ErrorType.TRANSIENT;
        }

        if (isTransactionError(error)) {
            return ErrorType.TRANSACTION;
        }

        if (isPermanentError(error)) {
            return ErrorType.PERMANENT;
        }

        if (isTimeoutError(error)) {
            return ErrorType.TIMEOUT;
        }

        return ErrorType.UNKNOWN;
    }

    private boolean isTransientError(Exception error) {
        // Erreurs réseau
        if (error instanceof MongoSocketException) {
            return true;
        }

        // Codes transitoires
        if (error instanceof MongoException) {
            MongoException mongoEx = (MongoException) error;
            return TRANSIENT_ERROR_CODES.contains(mongoEx.getCode());
        }

        return false;
    }

    private boolean isTransactionError(Exception error) {
        if (error instanceof MongoException) {
            MongoException mongoEx = (MongoException) error;
            return mongoEx.hasErrorLabel("TransientTransactionError") ||
                   mongoEx.hasErrorLabel("UnknownTransactionCommitResult");
        }
        return false;
    }

    private boolean isPermanentError(Exception error) {
        if (error instanceof MongoWriteException) {
            MongoWriteException writeEx = (MongoWriteException) error;
            // Duplicate key
            if (writeEx.getCode() == 11000) {
                return true;
            }
            // Authentication/Validation
            if (writeEx.getCode() == 13 || writeEx.getCode() == 121) {
                return true;
            }
        }
        return false;
    }

    private boolean isTimeoutError(Exception error) {
        return error instanceof MongoTimeoutException ||
               (error instanceof MongoException &&
                ((MongoException) error).getCode() == 50);
    }

    private long calculateDelay(int attempt) {
        long exponentialDelay = baseDelayMs * (long) Math.pow(2, attempt - 1);
        double jitter = Math.random() * 0.3 * exponentialDelay;
        long delay = Math.min((long) (exponentialDelay + jitter), maxDelayMs);
        return delay;
    }

    public <T> T retry(Supplier<T> operation, String context) throws Exception {
        Exception lastError = null;

        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                return operation.get();
            } catch (Exception error) {
                lastError = error;
                ErrorType errorType = classifyError(error);

                logger.warning(String.format(
                    "Operation failed (attempt %d/%d): %s - %s - Context: %s",
                    attempt, maxRetries, errorType, error.getMessage(), context
                ));

                // Erreur permanente
                if (errorType == ErrorType.PERMANENT) {
                    logger.severe("Permanent error, not retrying");
                    throw error;
                }

                // Dernier essai
                if (attempt == maxRetries) {
                    logger.severe("Max retries reached");
                    throw error;
                }

                // Attendre avant retry
                long delay = calculateDelay(attempt);
                logger.info(String.format("Retrying in %dms...", delay));
                Thread.sleep(delay);
            }
        }

        throw lastError;
    }
}

// Utilisation
MongoErrorHandler errorHandler = new MongoErrorHandler(3, 100, 5000);

// Exemple : Insert avec retry
public User insertUserWithRetry(User user) throws Exception {
    return errorHandler.retry(() -> {
        MongoCollection<Document> collection = database.getCollection("users");
        Document doc = new Document("name", user.getName())
                          .append("email", user.getEmail());
        collection.insertOne(doc);
        return user;
    }, "insertUser:" + user.getEmail());
}
```

### C# / .NET

```csharp
using MongoDB.Driver;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;

public class MongoErrorHandler
{
    private readonly int _maxRetries;
    private readonly int _baseDelayMs;
    private readonly int _maxDelayMs;
    private readonly ILogger _logger;

    private static readonly HashSet<int> TransientErrorCodes = new()
    {
        11600, 11602, 13436, 189, 91
    };

    public enum ErrorType
    {
        Transient,
        Transaction,
        Permanent,
        Timeout,
        Unknown
    }

    public MongoErrorHandler(int maxRetries = 3, int baseDelayMs = 100,
                            int maxDelayMs = 5000, ILogger logger = null)
    {
        _maxRetries = maxRetries;
        _baseDelayMs = baseDelayMs;
        _maxDelayMs = maxDelayMs;
        _logger = logger;
    }

    public ErrorType ClassifyError(Exception error)
    {
        if (IsTransientError(error)) return ErrorType.Transient;
        if (IsTransactionError(error)) return ErrorType.Transaction;
        if (IsPermanentError(error)) return ErrorType.Permanent;
        if (IsTimeoutError(error)) return ErrorType.Timeout;
        return ErrorType.Unknown;
    }

    private bool IsTransientError(Exception error)
    {
        // Erreurs réseau
        if (error is MongoConnectionException ||
            error is MongoNodeIsRecoveringException)
            return true;

        // Codes transitoires
        if (error is MongoCommandException cmdEx)
        {
            return TransientErrorCodes.Contains(cmdEx.Code);
        }

        return false;
    }

    private bool IsTransactionError(Exception error)
    {
        if (error is MongoException mongoEx)
        {
            return mongoEx.HasErrorLabel("TransientTransactionError") ||
                   mongoEx.HasErrorLabel("UnknownTransactionCommitResult");
        }
        return false;
    }

    private bool IsPermanentError(Exception error)
    {
        if (error is MongoWriteException writeEx)
        {
            // Duplicate key
            if (writeEx.WriteError?.Category == ServerErrorCategory.DuplicateKey)
                return true;

            // Authentication/Validation
            if (writeEx.WriteError?.Code == 13 || writeEx.WriteError?.Code == 121)
                return true;
        }

        return false;
    }

    private bool IsTimeoutError(Exception error)
    {
        return error is TimeoutException ||
               (error is MongoCommandException cmdEx && cmdEx.Code == 50);
    }

    private int CalculateDelay(int attempt)
    {
        var exponentialDelay = _baseDelayMs * Math.Pow(2, attempt - 1);
        var jitter = new Random().NextDouble() * 0.3 * exponentialDelay;
        var delay = (int)Math.Min(exponentialDelay + jitter, _maxDelayMs);
        return delay;
    }

    public async Task<T> RetryAsync<T>(
        Func<Task<T>> operation,
        string context = "",
        CancellationToken cancellationToken = default)
    {
        Exception lastError = null;

        for (int attempt = 1; attempt <= _maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception error)
            {
                lastError = error;
                var errorType = ClassifyError(error);

                _logger?.LogWarning(
                    "Operation failed (attempt {Attempt}/{MaxRetries}): {ErrorType} - {Message} - Context: {Context}",
                    attempt, _maxRetries, errorType, error.Message, context
                );

                // Erreur permanente
                if (errorType == ErrorType.Permanent)
                {
                    _logger?.LogError("Permanent error, not retrying");
                    throw;
                }

                // Dernier essai
                if (attempt == _maxRetries)
                {
                    _logger?.LogError("Max retries reached");
                    throw;
                }

                // Attendre avant retry
                var delay = CalculateDelay(attempt);
                _logger?.LogInformation("Retrying in {Delay}ms...", delay);
                await Task.Delay(delay, cancellationToken);
            }
        }

        throw lastError;
    }
}

// Utilisation
var errorHandler = new MongoErrorHandler(
    maxRetries: 3,
    baseDelayMs: 100,
    maxDelayMs: 5000,
    logger: logger
);

// Exemple : Insert avec retry
public async Task<User> InsertUserWithRetryAsync(User user)
{
    return await errorHandler.RetryAsync(async () =>
    {
        var collection = database.GetCollection<User>("users");
        await collection.InsertOneAsync(user);
        return user;
    }, $"insertUser:{user.Email}");
}
```

### Go

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "math"
    "math/rand"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
)

type ErrorType string

const (
    ErrorTypeTransient   ErrorType = "TRANSIENT"
    ErrorTypeTransaction ErrorType = "TRANSACTION"
    ErrorTypePermanent   ErrorType = "PERMANENT"
    ErrorTypeTimeout     ErrorType = "TIMEOUT"
    ErrorTypeUnknown     ErrorType = "UNKNOWN"
)

type MongoErrorHandler struct {
    MaxRetries  int
    BaseDelayMs int
    MaxDelayMs  int
}

func NewMongoErrorHandler(maxRetries, baseDelayMs, maxDelayMs int) *MongoErrorHandler {
    return &MongoErrorHandler{
        MaxRetries:  maxRetries,
        BaseDelayMs: baseDelayMs,
        MaxDelayMs:  maxDelayMs,
    }
}

func (h *MongoErrorHandler) ClassifyError(err error) ErrorType {
    if h.isTransientError(err) {
        return ErrorTypeTransient
    }

    if h.isTransactionError(err) {
        return ErrorTypeTransaction
    }

    if h.isPermanentError(err) {
        return ErrorTypePermanent
    }

    if h.isTimeoutError(err) {
        return ErrorTypeTimeout
    }

    return ErrorTypeUnknown
}

func (h *MongoErrorHandler) isTransientError(err error) bool {
    // Erreurs réseau
    if mongo.IsNetworkError(err) {
        return true
    }

    // Server selection timeout
    var serverSelErr mongo.ServerSelectionError
    if errors.As(err, &serverSelErr) {
        return true
    }

    return false
}

func (h *MongoErrorHandler) isTransactionError(err error) bool {
    cmdErr, ok := err.(mongo.CommandError)
    if !ok {
        return false
    }

    return cmdErr.HasErrorLabel("TransientTransactionError") ||
           cmdErr.HasErrorLabel("UnknownTransactionCommitResult")
}

func (h *MongoErrorHandler) isPermanentError(err error) bool {
    writeErr, ok := err.(mongo.WriteException)
    if ok {
        for _, we := range writeErr.WriteErrors {
            // Duplicate key
            if we.Code == 11000 {
                return true
            }
            // Authentication/Validation
            if we.Code == 13 || we.Code == 121 {
                return true
            }
        }
    }

    return false
}

func (h *MongoErrorHandler) isTimeoutError(err error) bool {
    if errors.Is(err, context.DeadlineExceeded) {
        return true
    }

    cmdErr, ok := err.(mongo.CommandError)
    if ok && cmdErr.Code == 50 {
        return true
    }

    return false
}

func (h *MongoErrorHandler) calculateDelay(attempt int) time.Duration {
    exponentialDelay := float64(h.BaseDelayMs) * math.Pow(2, float64(attempt-1))
    jitter := rand.Float64() * 0.3 * exponentialDelay
    delay := math.Min(exponentialDelay+jitter, float64(h.MaxDelayMs))
    return time.Duration(delay) * time.Millisecond
}

func (h *MongoErrorHandler) Retry(ctx context.Context, operation func() error, operationName string) error {
    var lastErr error

    for attempt := 1; attempt <= h.MaxRetries; attempt++ {
        err := operation()
        if err == nil {
            return nil
        }

        lastErr = err
        errorType := h.ClassifyError(err)

        fmt.Printf("Operation failed (attempt %d/%d): %s - %s - %s\n",
            attempt, h.MaxRetries, operationName, errorType, err.Error())

        // Erreur permanente
        if errorType == ErrorTypePermanent {
            fmt.Println("Permanent error, not retrying")
            return err
        }

        // Dernier essai
        if attempt == h.MaxRetries {
            fmt.Println("Max retries reached")
            return err
        }

        // Attendre avant retry
        delay := h.calculateDelay(attempt)
        fmt.Printf("Retrying in %v...\n", delay)

        select {
        case <-time.After(delay):
            // Continue
        case <-ctx.Done():
            return ctx.Err()
        }
    }

    return lastErr
}

// Utilisation
func main() {
    errorHandler := NewMongoErrorHandler(3, 100, 5000)

    // Exemple : Insert avec retry
    err := errorHandler.Retry(context.Background(), func() error {
        collection := client.Database("mydb").Collection("users")
        _, err := collection.InsertOne(context.Background(), user)
        return err
    }, "insertUser")

    if err != nil {
        log.Fatal(err)
    }
}
```

## Circuit Breaker

Le pattern Circuit Breaker prévient les tentatives répétées d'opérations qui échouent systématiquement.

```javascript
// Circuit Breaker pattern
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000; // 1 minute
    this.monitoringPeriod = options.monitoringPeriod || 10000; // 10 secondes

    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.nextAttempt = null;
  }

  async execute(operation) {
    // Circuit ouvert : rejeter immédiatement
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }

      // Essayer de fermer le circuit
      this.state = 'HALF_OPEN';
      this.successCount = 0;
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;

    if (this.state === 'HALF_OPEN') {
      this.successCount++;

      // Plusieurs succès consécutifs : fermer le circuit
      if (this.successCount >= 3) {
        this.state = 'CLOSED';
        console.log('Circuit breaker: HALF_OPEN -> CLOSED');
      }
    }
  }

  onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();

    if (this.state === 'HALF_OPEN') {
      // Échec en HALF_OPEN : rouvrir le circuit
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.log('Circuit breaker: HALF_OPEN -> OPEN');
    } else if (this.failures >= this.failureThreshold) {
      // Trop d'échecs : ouvrir le circuit
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.log('Circuit breaker: CLOSED -> OPEN');
    }
  }

  getState() {
    return {
      state: this.state,
      failures: this.failures,
      successCount: this.successCount,
      lastFailureTime: this.lastFailureTime
    };
  }
}

// Utilisation
const circuitBreaker = new CircuitBreaker({
  failureThreshold: 5,
  resetTimeout: 60000,
  monitoringPeriod: 10000
});

async function findUserWithCircuitBreaker(userId) {
  return await circuitBreaker.execute(async () => {
    const collection = db.collection('users');
    return await collection.findOne({ _id: userId });
  });
}
```

## Stratégies avancées

### Retry avec backoff exponentiel et jitter

```javascript
class ExponentialBackoffRetry {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 5;
    this.baseDelay = options.baseDelay || 100;
    this.maxDelay = options.maxDelay || 10000;
    this.jitterFactor = options.jitterFactor || 0.3; // ±30%
  }

  calculateDelay(attempt) {
    // Exponential: delay = baseDelay * 2^(attempt-1)
    const exponential = this.baseDelay * Math.pow(2, attempt - 1);

    // Jitter: ±jitterFactor%
    const jitterRange = exponential * this.jitterFactor;
    const jitter = (Math.random() * 2 - 1) * jitterRange;

    // Cap au maxDelay
    const delay = Math.min(exponential + jitter, this.maxDelay);

    return Math.max(0, delay);
  }

  async retry(operation) {
    let lastError;

    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;

        if (attempt === this.maxRetries) {
          throw error;
        }

        const delay = this.calculateDelay(attempt);
        console.log(`Attempt ${attempt} failed, retrying in ${delay.toFixed(0)}ms`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }

    throw lastError;
  }
}

// Exemple de progression du backoff
const backoff = new ExponentialBackoffRetry({
  maxRetries: 5,
  baseDelay: 100,
  maxDelay: 10000,
  jitterFactor: 0.3
});

// Tentative 1: échoue, attend ~100ms (70-130ms avec jitter)
// Tentative 2: échoue, attend ~200ms (140-260ms avec jitter)
// Tentative 3: échoue, attend ~400ms (280-520ms avec jitter)
// Tentative 4: échoue, attend ~800ms (560-1040ms avec jitter)
// Tentative 5: échoue, lance l'erreur
```

### Retry conditionnel

```javascript
class ConditionalRetry {
  constructor(errorHandler) {
    this.errorHandler = errorHandler;
  }

  async retry(operation, options = {}) {
    const {
      retryOn = (error) => true,
      maxRetries = 3,
      onRetry = (error, attempt) => {},
      shouldRetry = (error, attempt) => true
    } = options;

    let lastError;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;

        // Vérifier si on doit retry cette erreur
        if (!retryOn(error)) {
          throw error;
        }

        // Callback personnalisé
        onRetry(error, attempt);

        // Décision custom de retry
        if (!shouldRetry(error, attempt)) {
          throw error;
        }

        // Dernier essai
        if (attempt === maxRetries) {
          throw error;
        }

        const delay = this.errorHandler.calculateDelay(attempt);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }

    throw lastError;
  }
}

// Utilisation
const conditionalRetry = new ConditionalRetry(errorHandler);

// Retry seulement sur timeouts
await conditionalRetry.retry(
  () => collection.findOne({ _id: userId }),
  {
    retryOn: (error) => errorHandler.isTimeoutError(error),
    maxRetries: 5,
    onRetry: (error, attempt) => {
      console.log(`Timeout on attempt ${attempt}, retrying...`);
      metrics.incrementCounter('mongodb.retries.timeout');
    }
  }
);

// Retry avec limite de temps global
await conditionalRetry.retry(
  () => collection.aggregate(pipeline).toArray(),
  {
    maxRetries: 10,
    shouldRetry: (error, attempt) => {
      const elapsed = Date.now() - startTime;
      return elapsed < 30000; // Max 30 secondes au total
    }
  }
);
```

### Retry avec fallback

```javascript
class RetryWithFallback {
  constructor(primaryHandler, fallbackHandler) {
    this.primaryHandler = primaryHandler;
    this.fallbackHandler = fallbackHandler;
  }

  async execute(primaryOp, fallbackOp) {
    try {
      return await this.primaryHandler.retry(primaryOp);
    } catch (primaryError) {
      console.log('Primary operation failed, trying fallback...');

      try {
        return await this.fallbackHandler.retry(fallbackOp);
      } catch (fallbackError) {
        console.error('Both primary and fallback failed');
        throw new AggregateError(
          [primaryError, fallbackError],
          'All strategies failed'
        );
      }
    }
  }
}

// Utilisation
const retryWithFallback = new RetryWithFallback(
  new MongoErrorHandler({ maxRetries: 3 }),
  new MongoErrorHandler({ maxRetries: 2 })
);

// Primary: Query normale, Fallback: Query simplifiée
const result = await retryWithFallback.execute(
  // Primary
  () => collection.aggregate([
    { $match: { status: 'active' } },
    { $lookup: { from: 'orders', ... } },
    { $sort: { created_at: -1 } },
    { $limit: 100 }
  ]).toArray(),

  // Fallback
  () => collection.find({ status: 'active' })
                   .sort({ created_at: -1 })
                   .limit(100)
                   .toArray()
);
```

## Monitoring et métriques

```javascript
class RetryMetrics {
  constructor() {
    this.metrics = {
      totalRetries: 0,
      successAfterRetry: 0,
      failureAfterRetry: 0,
      byErrorType: {},
      byOperation: {},
      retryDurations: []
    };
  }

  recordRetry(errorType, operation, success, duration) {
    this.metrics.totalRetries++;

    if (success) {
      this.metrics.successAfterRetry++;
    } else {
      this.metrics.failureAfterRetry++;
    }

    // Par type d'erreur
    if (!this.metrics.byErrorType[errorType]) {
      this.metrics.byErrorType[errorType] = { count: 0, successes: 0 };
    }
    this.metrics.byErrorType[errorType].count++;
    if (success) {
      this.metrics.byErrorType[errorType].successes++;
    }

    // Par opération
    if (!this.metrics.byOperation[operation]) {
      this.metrics.byOperation[operation] = { count: 0, successes: 0 };
    }
    this.metrics.byOperation[operation].count++;
    if (success) {
      this.metrics.byOperation[operation].successes++;
    }

    // Durée
    this.metrics.retryDurations.push(duration);
    if (this.metrics.retryDurations.length > 1000) {
      this.metrics.retryDurations.shift();
    }
  }

  getStats() {
    const durations = this.metrics.retryDurations;
    return {
      ...this.metrics,
      avgRetryDuration: durations.length > 0
        ? durations.reduce((a, b) => a + b, 0) / durations.length
        : 0,
      successRate: this.metrics.totalRetries > 0
        ? (this.metrics.successAfterRetry / this.metrics.totalRetries * 100).toFixed(2)
        : 0
    };
  }

  reset() {
    this.metrics = {
      totalRetries: 0,
      successAfterRetry: 0,
      failureAfterRetry: 0,
      byErrorType: {},
      byOperation: {},
      retryDurations: []
    };
  }
}

// Intégration dans l'error handler
class MongoErrorHandlerWithMetrics extends MongoErrorHandler {
  constructor(options = {}) {
    super(options);
    this.metrics = new RetryMetrics();
  }

  async retry(operation, context = {}) {
    const startTime = Date.now();
    let success = false;
    let errorType = 'UNKNOWN';

    try {
      const result = await super.retry(operation, context);
      success = true;
      return result;
    } catch (error) {
      errorType = this.classifyError(error);
      throw error;
    } finally {
      const duration = Date.now() - startTime;
      this.metrics.recordRetry(
        errorType,
        context.operation || 'unknown',
        success,
        duration
      );
    }
  }

  getMetrics() {
    return this.metrics.getStats();
  }
}

// Exposition des métriques
app.get('/metrics/retries', (req, res) => {
  const stats = errorHandler.getMetrics();
  res.json(stats);
});
```

## Bonnes pratiques

### ✅ DO - À faire

```javascript
// 1. Activer le retry automatique du driver
const client = new MongoClient(uri, {
  retryWrites: true,
  retryReads: true
});

// 2. Classifier les erreurs avant retry
if (errorHandler.isPermanentError(error)) {
  throw error; // Ne pas retry
}

// 3. Utiliser exponential backoff + jitter
const delay = baseDelay * Math.pow(2, attempt - 1) + randomJitter();

// 4. Limiter le nombre de retries
const MAX_RETRIES = 3; // Pas trop de retries

// 5. Logger les retries pour monitoring
logger.warn('Retry attempt', { attempt, error, context });

// 6. Implémenter des timeouts
const timeout = setTimeout(() => {
  reject(new Error('Operation timeout'));
}, 30000);

// 7. Utiliser un circuit breaker pour erreurs répétées
const result = await circuitBreaker.execute(operation);

// 8. Monitorer les métriques de retry
metrics.recordRetry(errorType, success, duration);

// 9. Tester les scénarios d'erreur
// Tests avec erreurs simulées

// 10. Documenter les stratégies de retry
// Pour chaque opération critique
```

### ❌ DON'T - À éviter

```javascript
// ❌ 1. Retry sur toutes les erreurs
try {
  await operation();
} catch (error) {
  await operation(); // Même erreur permanente !
}

// ❌ 2. Retry sans délai
for (let i = 0; i < 100; i++) {
  try {
    await operation();
    break;
  } catch (error) {
    // Immédiatement sans pause
  }
}

// ❌ 3. Trop de retries
const MAX_RETRIES = 100; // Trop !

// ❌ 4. Pas de timeout
await retryForever(operation); // Dangereux

// ❌ 5. Ignorer les erreurs permanentes
// Duplicate key continuera à échouer

// ❌ 6. Pas de monitoring
// Impossible de détecter les problèmes

// ❌ 7. Délai fixe sans backoff
await sleep(1000); // Toujours 1s

// ❌ 8. Retry synchrone bloquant
while (true) {
  try { return operation(); }
  catch { Thread.sleep(1000); } // Bloque tout
}

// ❌ 9. Pas de circuit breaker
// Surcharge le serveur défaillant

// ❌ 10. Pas de contexte dans les logs
logger.error('Retry failed'); // Pas d'info
```

## Conclusion

La gestion des erreurs et les stratégies de retry sont essentielles pour la résilience :

1. **Classifier les erreurs** : Transitoires vs permanentes
2. **Retry intelligent** : Exponential backoff + jitter
3. **Limites appropriées** : maxRetries et timeouts
4. **Circuit breaker** : Protéger contre surcharge
5. **Monitoring** : Métriques et alertes
6. **Driver retry** : Activer retryWrites/retryReads

**Configuration recommandée** :
- maxRetries: 3-5
- baseDelay: 100ms
- maxDelay: 5000ms
- jitterFactor: 0.3
- Circuit breaker activé
- Monitoring complet

---


⏭️ [ODM et ORM](/15-drivers-integration-applicative/12-odm-orm.md)
