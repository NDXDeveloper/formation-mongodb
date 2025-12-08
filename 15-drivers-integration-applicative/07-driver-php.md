🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.7 Driver PHP

## Introduction

Le driver PHP MongoDB se compose de deux parties : une **extension C** (mongodb) pour les performances et une **library PHP** (mongodb/mongodb) qui fournit une API de haut niveau. Cette architecture offre d'excellentes performances tout en gardant une API PHP idiomatique. Cette section couvre l'utilisation moderne avec **PHP 8.1+** et l'intégration avec les frameworks populaires.

## Installation et configuration

### Prérequis

```bash
# PHP 8.1 ou supérieur
php -v

# Extension MongoDB (requis)
# Ubuntu/Debian
sudo apt-get install php-mongodb

# macOS (avec Homebrew)
brew install php
pecl install mongodb

# Windows
# Télécharger depuis https://pecl.php.net/package/mongodb
# Ajouter extension=mongodb.so (ou .dll) dans php.ini
```

### Installation avec Composer

```bash
# Library PHP (wrapper de haut niveau)
composer require mongodb/mongodb

# Pour Laravel
composer require mongodb/laravel-mongodb

# Pour Symfony
composer require mongodb/mongodb
composer require doctrine/mongodb-odm-bundle
```

### composer.json

```json
{
    "name": "myapp/mongodb-app",
    "description": "Application with MongoDB",
    "type": "project",
    "require": {
        "php": "^8.1",
        "mongodb/mongodb": "^1.18",
        "vlucas/phpdotenv": "^5.5",
        "monolog/monolog": "^3.5"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "phpstan/phpstan": "^1.10"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

### Structure de projet

```
myapp/
├── composer.json
├── .env
├── .env.example
├── public/
│   └── index.php
├── src/
│   ├── Config/
│   │   └── Database.php
│   ├── Models/
│   │   ├── User.php
│   │   └── UserStatus.php
│   ├── Repository/
│   │   ├── BaseRepository.php
│   │   └── UserRepository.php
│   ├── Service/
│   │   └── UserService.php
│   ├── Controller/
│   │   └── UserController.php
│   └── bootstrap.php
└── tests/
```

## Configuration

### .env

```env
MONGODB_URI=mongodb://localhost:27017
MONGODB_DATABASE=myapp
MONGODB_MAX_POOL_SIZE=50
MONGODB_MIN_POOL_SIZE=10
MONGODB_CONNECT_TIMEOUT_MS=10000
MONGODB_SOCKET_TIMEOUT_MS=45000
```

### Configuration de base

```php
<?php
// src/Config/Database.php

namespace App\Config;

use MongoDB\Client;
use MongoDB\Database as MongoDatabase;
use Psr\Log\LoggerInterface;

class Database
{
    private static ?Client $client = null;
    private static ?MongoDatabase $database = null;

    public static function getClient(): Client
    {
        if (self::$client === null) {
            $uri = $_ENV['MONGODB_URI'] ?? 'mongodb://localhost:27017';

            $uriOptions = [
                'maxPoolSize' => (int)($_ENV['MONGODB_MAX_POOL_SIZE'] ?? 50),
                'minPoolSize' => (int)($_ENV['MONGODB_MIN_POOL_SIZE'] ?? 10),
                'connectTimeoutMS' => (int)($_ENV['MONGODB_CONNECT_TIMEOUT_MS'] ?? 10000),
                'socketTimeoutMS' => (int)($_ENV['MONGODB_SOCKET_TIMEOUT_MS'] ?? 45000),
                'serverSelectionTimeoutMS' => 5000,
                'retryWrites' => true,
                'retryReads' => true,
            ];

            $driverOptions = [];

            self::$client = new Client($uri, $uriOptions, $driverOptions);

            // Vérifier la connexion
            try {
                self::$client->selectDatabase('admin')->command(['ping' => 1]);
                error_log('✅ Connected to MongoDB');
            } catch (\Exception $e) {
                error_log('❌ Failed to connect to MongoDB: ' . $e->getMessage());
                throw $e;
            }
        }

        return self::$client;
    }

    public static function getDatabase(): MongoDatabase
    {
        if (self::$database === null) {
            $databaseName = $_ENV['MONGODB_DATABASE'] ?? 'myapp';
            self::$database = self::getClient()->selectDatabase($databaseName);
        }

        return self::$database;
    }

    public static function close(): void
    {
        self::$client = null;
        self::$database = null;
    }
}
```

### Configuration avancée avec logger

```php
<?php
// src/Config/DatabaseAdvanced.php

namespace App\Config;

use MongoDB\Client;
use MongoDB\Database as MongoDatabase;
use MongoDB\Driver\Monitoring\CommandSubscriber;
use MongoDB\Driver\Monitoring\CommandStartedEvent;
use MongoDB\Driver\Monitoring\CommandSucceededEvent;
use MongoDB\Driver\Monitoring\CommandFailedEvent;
use Psr\Log\LoggerInterface;

class MongoDBLogger implements CommandSubscriber
{
    public function __construct(private LoggerInterface $logger)
    {
    }

    public function commandStarted(CommandStartedEvent $event): void
    {
        $this->logger->debug('MongoDB command started', [
            'command' => $event->getCommandName(),
            'request_id' => $event->getRequestId(),
        ]);
    }

    public function commandSucceeded(CommandSucceededEvent $event): void
    {
        $this->logger->debug('MongoDB command succeeded', [
            'command' => $event->getCommandName(),
            'duration_ms' => $event->getDurationMicros() / 1000,
        ]);
    }

    public function commandFailed(CommandFailedEvent $event): void
    {
        $this->logger->error('MongoDB command failed', [
            'command' => $event->getCommandName(),
            'error' => $event->getError()->getMessage(),
        ]);
    }
}

class DatabaseAdvanced
{
    private Client $client;
    private MongoDatabase $database;

    public function __construct(
        private array $config,
        private ?LoggerInterface $logger = null
    ) {
        $this->connect();
    }

    private function connect(): void
    {
        $uri = $this->config['uri'] ?? 'mongodb://localhost:27017';

        $uriOptions = [
            'maxPoolSize' => $this->config['max_pool_size'] ?? 50,
            'minPoolSize' => $this->config['min_pool_size'] ?? 10,
            'connectTimeoutMS' => $this->config['connect_timeout_ms'] ?? 10000,
            'socketTimeoutMS' => $this->config['socket_timeout_ms'] ?? 45000,
            'serverSelectionTimeoutMS' => 5000,
            'retryWrites' => true,
            'retryReads' => true,
        ];

        $this->client = new Client($uri, $uriOptions);

        // Ajouter le monitoring si logger fourni
        if ($this->logger !== null) {
            \MongoDB\Driver\Monitoring\addSubscriber(
                new MongoDBLogger($this->logger)
            );
        }

        $databaseName = $this->config['database'] ?? 'myapp';
        $this->database = $this->client->selectDatabase($databaseName);

        // Ping pour vérifier
        try {
            $this->database->command(['ping' => 1]);
            $this->logger?->info('✅ Connected to MongoDB');
        } catch (\Exception $e) {
            $this->logger?->error('❌ Failed to connect to MongoDB', [
                'error' => $e->getMessage()
            ]);
            throw $e;
        }
    }

    public function getClient(): Client
    {
        return $this->client;
    }

    public function getDatabase(): MongoDatabase
    {
        return $this->database;
    }
}
```

## Modèles de données

### Énumération UserStatus (PHP 8.1+)

```php
<?php
// src/Models/UserStatus.php

namespace App\Models;

enum UserStatus: string
{
    case ACTIVE = 'active';
    case INACTIVE = 'inactive';
    case SUSPENDED = 'suspended';
    case DELETED = 'deleted';

    public function label(): string
    {
        return match($this) {
            self::ACTIVE => 'Actif',
            self::INACTIVE => 'Inactif',
            self::SUSPENDED => 'Suspendu',
            self::DELETED => 'Supprimé',
        };
    }
}
```

### Modèle User avec propriétés typées

```php
<?php
// src/Models/User.php

namespace App\Models;

use MongoDB\BSON\ObjectId;
use MongoDB\BSON\UTCDateTime;

class User
{
    public function __construct(
        public ?ObjectId $id = null,
        public string $name = '',
        public string $email = '',
        public ?int $age = null,
        public UserStatus $status = UserStatus::ACTIVE,
        public array $roles = ['user'],
        public ?array $preferences = null,
        public ?UTCDateTime $createdAt = null,
        public ?UTCDateTime $updatedAt = null,
        public ?UTCDateTime $lastLogin = null,
        public int $loginCount = 0,
    ) {
        $this->createdAt ??= new UTCDateTime();
        $this->updatedAt ??= new UTCDateTime();
    }

    public static function fromArray(array $data): self
    {
        return new self(
            id: $data['_id'] ?? null,
            name: $data['name'] ?? '',
            email: $data['email'] ?? '',
            age: $data['age'] ?? null,
            status: isset($data['status'])
                ? UserStatus::from($data['status'])
                : UserStatus::ACTIVE,
            roles: $data['roles'] ?? ['user'],
            preferences: $data['preferences'] ?? null,
            createdAt: $data['created_at'] ?? null,
            updatedAt: $data['updated_at'] ?? null,
            lastLogin: $data['last_login'] ?? null,
            loginCount: $data['login_count'] ?? 0,
        );
    }

    public function toArray(): array
    {
        $data = [
            'name' => $this->name,
            'email' => $this->email,
            'status' => $this->status->value,
            'roles' => $this->roles,
            'created_at' => $this->createdAt,
            'updated_at' => $this->updatedAt,
            'login_count' => $this->loginCount,
        ];

        if ($this->id !== null) {
            $data['_id'] = $this->id;
        }

        if ($this->age !== null) {
            $data['age'] = $this->age;
        }

        if ($this->preferences !== null) {
            $data['preferences'] = $this->preferences;
        }

        if ($this->lastLogin !== null) {
            $data['last_login'] = $this->lastLogin;
        }

        return $data;
    }

    public function toPublicArray(): array
    {
        return [
            'id' => (string)$this->id,
            'name' => $this->name,
            'email' => $this->email,
            'status' => $this->status->value,
            'created_at' => $this->createdAt?->toDateTime()->format('c'),
        ];
    }

    public function incrementLogin(): void
    {
        $this->loginCount++;
        $this->lastLogin = new UTCDateTime();
        $this->updatedAt = new UTCDateTime();
    }
}
```

### DTOs

```php
<?php
// src/Models/CreateUserDTO.php

namespace App\Models;

readonly class CreateUserDTO
{
    public function __construct(
        public string $name,
        public string $email,
        public ?int $age,
        public string $password,
        public array $roles = ['user'],
    ) {
    }

    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'] ?? throw new \InvalidArgumentException('Name is required'),
            email: $data['email'] ?? throw new \InvalidArgumentException('Email is required'),
            age: $data['age'] ?? null,
            password: $data['password'] ?? throw new \InvalidArgumentException('Password is required'),
            roles: $data['roles'] ?? ['user'],
        );
    }

    public function validate(): array
    {
        $errors = [];

        if (strlen($this->name) < 2 || strlen($this->name) > 100) {
            $errors['name'] = 'Name must be between 2 and 100 characters';
        }

        if (!filter_var($this->email, FILTER_VALIDATE_EMAIL)) {
            $errors['email'] = 'Invalid email format';
        }

        if ($this->age !== null && ($this->age < 18 || $this->age > 120)) {
            $errors['age'] = 'Age must be between 18 and 120';
        }

        if (strlen($this->password) < 8) {
            $errors['password'] = 'Password must be at least 8 characters';
        }

        return $errors;
    }
}
```

```php
<?php
// src/Models/UpdateUserDTO.php

namespace App\Models;

readonly class UpdateUserDTO
{
    public function __construct(
        public ?string $name = null,
        public ?string $email = null,
        public ?int $age = null,
    ) {
    }

    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'] ?? null,
            email: $data['email'] ?? null,
            age: $data['age'] ?? null,
        );
    }

    public function toUpdateArray(): array
    {
        $updates = [];

        if ($this->name !== null) {
            $updates['name'] = $this->name;
        }

        if ($this->email !== null) {
            $updates['email'] = $this->email;
        }

        if ($this->age !== null) {
            $updates['age'] = $this->age;
        }

        return $updates;
    }
}
```

## Repository Pattern

### Repository de base

```php
<?php
// src/Repository/BaseRepository.php

namespace App\Repository;

use MongoDB\Collection;
use MongoDB\BSON\ObjectId;

abstract class BaseRepository
{
    protected Collection $collection;

    public function __construct(Collection $collection)
    {
        $this->collection = $collection;
    }

    public function findById(string|ObjectId $id): ?array
    {
        if (is_string($id)) {
            $id = new ObjectId($id);
        }

        $document = $this->collection->findOne(['_id' => $id]);
        return $document !== null ? (array)$document : null;
    }

    public function findOne(array $filter): ?array
    {
        $document = $this->collection->findOne($filter);
        return $document !== null ? (array)$document : null;
    }

    public function findMany(array $filter = [], array $options = []): array
    {
        return $this->collection->find($filter, $options)->toArray();
    }

    public function insertOne(array $document): ObjectId
    {
        $result = $this->collection->insertOne($document);
        return $result->getInsertedId();
    }

    public function updateOne(array $filter, array $update): bool
    {
        $result = $this->collection->updateOne($filter, $update);
        return $result->getModifiedCount() > 0;
    }

    public function updateById(string|ObjectId $id, array $update): bool
    {
        if (is_string($id)) {
            $id = new ObjectId($id);
        }

        return $this->updateOne(['_id' => $id], $update);
    }

    public function deleteOne(array $filter): bool
    {
        $result = $this->collection->deleteOne($filter);
        return $result->getDeletedCount() > 0;
    }

    public function deleteById(string|ObjectId $id): bool
    {
        if (is_string($id)) {
            $id = new ObjectId($id);
        }

        return $this->deleteOne(['_id' => $id]);
    }

    public function count(array $filter = []): int
    {
        return $this->collection->countDocuments($filter);
    }

    public function exists(array $filter): bool
    {
        return $this->collection->countDocuments($filter, ['limit' => 1]) > 0;
    }
}
```

### Repository utilisateur

```php
<?php
// src/Repository/UserRepository.php

namespace App\Repository;

use App\Models\User;
use App\Models\UserStatus;
use MongoDB\BSON\ObjectId;
use MongoDB\BSON\UTCDateTime;
use MongoDB\Collection;

class UserRepository extends BaseRepository
{
    public function __construct(Collection $collection)
    {
        parent::__construct($collection);
        $this->ensureIndexes();
    }

    private function ensureIndexes(): void
    {
        // Index unique sur email
        $this->collection->createIndex(
            ['email' => 1],
            ['unique' => true]
        );

        // Index sur status
        $this->collection->createIndex(['status' => 1]);

        // Index composé
        $this->collection->createIndex([
            'status' => 1,
            'created_at' => -1,
        ]);

        error_log('Indexes created for users collection');
    }

    public function create(User $user): User
    {
        try {
            $data = $user->toArray();
            $id = $this->insertOne($data);
            $user->id = $id;
            return $user;
        } catch (\MongoDB\Driver\Exception\BulkWriteException $e) {
            if ($e->getCode() === 11000) {
                throw new \RuntimeException('Email already exists', 0, $e);
            }
            throw $e;
        }
    }

    public function findByEmail(string $email): ?User
    {
        $document = $this->findOne(['email' => $email]);
        return $document !== null ? User::fromArray($document) : null;
    }

    public function findActiveUsers(int $limit = 100, int $skip = 0): array
    {
        $documents = $this->findMany(
            ['status' => UserStatus::ACTIVE->value],
            [
                'limit' => $limit,
                'skip' => $skip,
                'sort' => ['created_at' => -1],
            ]
        );

        return array_map(fn($doc) => User::fromArray($doc), $documents);
    }

    public function update(ObjectId $id, array $updates): bool
    {
        $updates['updated_at'] = new UTCDateTime();

        return $this->updateById($id, ['$set' => $updates]);
    }

    public function incrementLoginCount(ObjectId $id): void
    {
        $this->collection->updateOne(
            ['_id' => $id],
            [
                '$inc' => ['login_count' => 1],
                '$set' => [
                    'last_login' => new UTCDateTime(),
                    'updated_at' => new UTCDateTime(),
                ],
            ]
        );
    }

    public function search(string $searchTerm, int $limit = 20): array
    {
        $documents = $this->findMany(
            [
                '$or' => [
                    ['name' => ['$regex' => $searchTerm, '$options' => 'i']],
                    ['email' => ['$regex' => $searchTerm, '$options' => 'i']],
                ],
            ],
            ['limit' => $limit]
        );

        return array_map(fn($doc) => User::fromArray($doc), $documents);
    }

    public function findByRole(string $role): array
    {
        $documents = $this->findMany(['roles' => $role]);
        return array_map(fn($doc) => User::fromArray($doc), $documents);
    }

    public function bulkUpdateStatus(array $userIds, UserStatus $status): int
    {
        $objectIds = array_map(
            fn($id) => is_string($id) ? new ObjectId($id) : $id,
            $userIds
        );

        $result = $this->collection->updateMany(
            ['_id' => ['$in' => $objectIds]],
            [
                '$set' => [
                    'status' => $status->value,
                    'updated_at' => new UTCDateTime(),
                ],
            ]
        );

        return $result->getModifiedCount();
    }

    public function getStatistics(): array
    {
        // Total
        $total = $this->count();

        // Par status
        $byStatus = [];
        $pipeline = [
            ['$group' => [
                '_id' => '$status',
                'count' => ['$sum' => 1],
            ]],
        ];

        $cursor = $this->collection->aggregate($pipeline);
        foreach ($cursor as $doc) {
            $byStatus[$doc->_id] = $doc->count;
        }

        // Inscriptions récentes (30 derniers jours)
        $thirtyDaysAgo = new UTCDateTime(
            (new \DateTime('-30 days'))->getTimestamp() * 1000
        );
        $recentSignups = $this->count([
            'created_at' => ['$gte' => $thirtyDaysAgo],
        ]);

        return [
            'total' => $total,
            'by_status' => $byStatus,
            'recent_signups' => $recentSignups,
        ];
    }
}
```

## Service Layer

```php
<?php
// src/Service/UserService.php

namespace App\Service;

use App\Models\User;
use App\Models\UserStatus;
use App\Models\CreateUserDTO;
use App\Models\UpdateUserDTO;
use App\Repository\UserRepository;
use MongoDB\BSON\ObjectId;
use Psr\Log\LoggerInterface;

class UserService
{
    public function __construct(
        private UserRepository $repository,
        private ?LoggerInterface $logger = null,
    ) {
    }

    public function registerUser(CreateUserDTO $dto): User
    {
        // Valider
        $errors = $dto->validate();
        if (!empty($errors)) {
            throw new \InvalidArgumentException(
                'Validation failed: ' . json_encode($errors)
            );
        }

        // Vérifier si l'email existe
        $existing = $this->repository->findByEmail($dto->email);
        if ($existing !== null) {
            throw new \RuntimeException("Email {$dto->email} already registered");
        }

        // Créer l'utilisateur
        $user = new User(
            name: $dto->name,
            email: $dto->email,
            age: $dto->age,
            roles: $dto->roles,
        );

        // TODO: Hasher le mot de passe
        // $user->passwordHash = password_hash($dto->password, PASSWORD_ARGON2ID);

        $user = $this->repository->create($user);

        $this->logger?->info('New user registered', [
            'email' => $user->email,
            'id' => (string)$user->id,
        ]);

        return $user;
    }

    public function authenticateUser(string $email, string $password): User
    {
        $user = $this->repository->findByEmail($email);

        if ($user === null) {
            throw new \RuntimeException('Invalid credentials');
        }

        if ($user->status !== UserStatus::ACTIVE) {
            throw new \RuntimeException('Account is not active');
        }

        // TODO: Vérifier le mot de passe
        // if (!password_verify($password, $user->passwordHash)) {
        //     throw new \RuntimeException('Invalid credentials');
        // }

        // Incrémenter le compteur
        $this->repository->incrementLoginCount($user->id);

        $this->logger?->info('User authenticated', [
            'email' => $user->email,
            'id' => (string)$user->id,
        ]);

        return $user;
    }

    public function getUserById(string $id): User
    {
        $objectId = new ObjectId($id);
        $document = $this->repository->findById($objectId);

        if ($document === null) {
            throw new \RuntimeException('User not found');
        }

        return User::fromArray($document);
    }

    public function updateUser(string $id, UpdateUserDTO $dto): User
    {
        $objectId = new ObjectId($id);
        $updates = $dto->toUpdateArray();

        if (empty($updates)) {
            throw new \InvalidArgumentException('No updates provided');
        }

        $success = $this->repository->update($objectId, $updates);

        if (!$success) {
            throw new \RuntimeException('Failed to update user');
        }

        return $this->getUserById($id);
    }

    public function searchUsers(string $query, int $limit = 20): array
    {
        return $this->repository->search($query, $limit);
    }

    public function getDashboardStats(): array
    {
        return $this->repository->getStatistics();
    }

    public function suspendUser(string $id): bool
    {
        $objectId = new ObjectId($id);

        $success = $this->repository->update($objectId, [
            'status' => UserStatus::SUSPENDED->value,
        ]);

        if ($success) {
            $this->logger?->info('User suspended', ['id' => $id]);
        }

        return $success;
    }
}
```

## Opérations avancées

### Pagination

```php
<?php
// src/Repository/PaginationTrait.php

namespace App\Repository;

trait PaginationTrait
{
    public function findPaginated(
        array $filter = [],
        int $page = 1,
        int $pageSize = 20
    ): array {
        // Total
        $total = $this->collection->countDocuments($filter);

        // Skip
        $skip = ($page - 1) * $pageSize;

        // Documents
        $documents = $this->collection->find(
            $filter,
            [
                'limit' => $pageSize,
                'skip' => $skip,
                'sort' => ['created_at' => -1],
            ]
        )->toArray();

        $totalPages = (int)ceil($total / $pageSize);

        return [
            'data' => $documents,
            'total' => $total,
            'page' => $page,
            'page_size' => $pageSize,
            'total_pages' => $totalPages,
        ];
    }
}
```

### Bulk Operations

```php
<?php

function bulkUpdatePrices(Collection $collection, array $updates): int
{
    $bulkWrite = [];

    foreach ($updates as $update) {
        $bulkWrite[] = [
            'updateOne' => [
                ['_id' => new ObjectId($update['product_id'])],
                [
                    '$set' => [
                        'price.amount' => $update['new_price'],
                        'updated_at' => new UTCDateTime(),
                    ],
                ],
            ],
        ];
    }

    $result = $collection->bulkWrite($bulkWrite, ['ordered' => false]);

    return $result->getModifiedCount();
}
```

## Agrégations

```php
<?php
// src/Service/AnalyticsService.php

namespace App\Service;

use MongoDB\Collection;
use MongoDB\BSON\UTCDateTime;

class AnalyticsService
{
    public function __construct(
        private Collection $ordersCollection,
        private Collection $productsCollection,
    ) {
    }

    public function getSalesAnalytics(\DateTime $startDate, \DateTime $endDate): array
    {
        $pipeline = [
            // Filtrer par période
            [
                '$match' => [
                    'created_at' => [
                        '$gte' => new UTCDateTime($startDate),
                        '$lte' => new UTCDateTime($endDate),
                    ],
                    'status' => 'completed',
                ],
            ],

            // Dérouler les items
            ['$unwind' => '$items'],

            // Jointure avec products
            [
                '$lookup' => [
                    'from' => 'products',
                    'localField' => 'items.product_id',
                    'foreignField' => '_id',
                    'as' => 'product_details',
                ],
            ],

            ['$unwind' => '$product_details'],

            // Grouper par catégorie
            [
                '$group' => [
                    '_id' => '$product_details.category',
                    'total_sales' => [
                        '$sum' => [
                            '$multiply' => ['$items.quantity', '$items.price'],
                        ],
                    ],
                    'total_items' => ['$sum' => '$items.quantity'],
                    'avg_order_value' => ['$avg' => '$total_amount'],
                ],
            ],

            // Trier
            ['$sort' => ['total_sales' => -1]],
        ];

        $cursor = $this->ordersCollection->aggregate($pipeline);

        return [
            'period' => [
                'start' => $startDate->format('c'),
                'end' => $endDate->format('c'),
            ],
            'by_category' => iterator_to_array($cursor),
        ];
    }

    public function getProductDashboard(): array
    {
        $pipeline = [
            [
                '$facet' => [
                    'general_stats' => [
                        [
                            '$group' => [
                                '_id' => null,
                                'total_products' => ['$sum' => 1],
                                'avg_price' => ['$avg' => '$price.amount'],
                                'total_inventory' => ['$sum' => '$inventory.quantity'],
                            ],
                        ],
                    ],
                    'by_category' => [
                        [
                            '$group' => [
                                '_id' => '$category',
                                'count' => ['$sum' => 1],
                                'avg_price' => ['$avg' => '$price.amount'],
                            ],
                        ],
                        ['$sort' => ['count' => -1]],
                    ],
                    'top_products' => [
                        ['$sort' => ['reviews.rating' => -1]],
                        ['$limit' => 10],
                        [
                            '$project' => [
                                'name' => 1,
                                'price' => '$price.amount',
                                'rating' => '$reviews.rating',
                            ],
                        ],
                    ],
                ],
            ],
        ];

        $cursor = $this->productsCollection->aggregate($pipeline);
        $result = iterator_to_array($cursor);

        return $result[0] ?? [];
    }
}
```

## Transactions

```php
<?php
// src/Service/TransactionService.php

namespace App\Service;

use MongoDB\Client;
use MongoDB\BSON\ObjectId;
use MongoDB\BSON\UTCDateTime;

class TransactionService
{
    public function __construct(private Client $client)
    {
    }

    public function transferFunds(
        ObjectId $fromAccountId,
        ObjectId $toAccountId,
        float $amount
    ): void {
        $session = $this->client->startSession();

        try {
            $session->startTransaction();

            $accounts = $this->client->selectDatabase('myapp')
                ->selectCollection('accounts');

            // Débiter
            $debitResult = $accounts->updateOne(
                [
                    '_id' => $fromAccountId,
                    'balance' => ['$gte' => $amount],
                ],
                [
                    '$inc' => ['balance' => -$amount],
                    '$push' => [
                        'transactions' => [
                            'type' => 'debit',
                            'amount' => $amount,
                            'timestamp' => new UTCDateTime(),
                        ],
                    ],
                ],
                ['session' => $session]
            );

            if ($debitResult->getModifiedCount() === 0) {
                throw new \RuntimeException('Insufficient funds or account not found');
            }

            // Créditer
            $accounts->updateOne(
                ['_id' => $toAccountId],
                [
                    '$inc' => ['balance' => $amount],
                    '$push' => [
                        'transactions' => [
                            'type' => 'credit',
                            'amount' => $amount,
                            'timestamp' => new UTCDateTime(),
                        ],
                    ],
                ],
                ['session' => $session]
            );

            $session->commitTransaction();
            error_log('✅ Transaction completed successfully');

        } catch (\Exception $e) {
            $session->abortTransaction();
            error_log('❌ Transaction failed: ' . $e->getMessage());
            throw $e;
        } finally {
            $session->endSession();
        }
    }

    public function createOrderWithInventoryUpdate(array $orderData): ObjectId
    {
        $maxRetries = 3;
        $lastException = null;

        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            $session = $this->client->startSession();

            try {
                $session->startTransaction();

                $db = $this->client->selectDatabase('myapp');
                $orders = $db->selectCollection('orders');
                $products = $db->selectCollection('products');
                $users = $db->selectCollection('users');

                // 1. Réserver l'inventaire
                foreach ($orderData['items'] as $item) {
                    $result = $products->updateOne(
                        [
                            '_id' => new ObjectId($item['product_id']),
                            'inventory.quantity' => ['$gte' => $item['quantity']],
                        ],
                        [
                            '$inc' => ['inventory.quantity' => -$item['quantity']],
                            '$set' => ['updated_at' => new UTCDateTime()],
                        ],
                        ['session' => $session]
                    );

                    if ($result->getModifiedCount() === 0) {
                        throw new \RuntimeException(
                            "Product {$item['product_id']} out of stock"
                        );
                    }
                }

                // 2. Créer la commande
                $order = [
                    'user_id' => new ObjectId($orderData['user_id']),
                    'items' => $orderData['items'],
                    'status' => 'pending',
                    'total_amount' => $orderData['total_amount'],
                    'created_at' => new UTCDateTime(),
                    'updated_at' => new UTCDateTime(),
                ];

                $orderResult = $orders->insertOne($order, ['session' => $session]);
                $orderId = $orderResult->getInsertedId();

                // 3. Mettre à jour les stats utilisateur
                $users->updateOne(
                    ['_id' => new ObjectId($orderData['user_id'])],
                    [
                        '$inc' => ['stats.total_orders' => 1],
                        '$set' => ['stats.last_order_date' => new UTCDateTime()],
                    ],
                    ['session' => $session]
                );

                $session->commitTransaction();
                error_log("✅ Order created: {$orderId}");

                return $orderId;

            } catch (\MongoDB\Driver\Exception\RuntimeException $e) {
                $session->abortTransaction();
                $lastException = $e;

                // Retry sur erreurs transitoires
                if (strpos($e->getMessage(), 'TransientTransactionError') !== false
                    && $attempt < $maxRetries) {
                    error_log("Transaction failed (attempt {$attempt}), retrying...");
                    usleep(100000 * $attempt); // 100ms * attempt
                    continue;
                }

                throw $e;

            } finally {
                $session->endSession();
            }
        }

        throw new \RuntimeException(
            "Transaction failed after {$maxRetries} attempts",
            0,
            $lastException
        );
    }
}
```

## Intégration Laravel

### Configuration Laravel

```php
<?php
// config/database.php

return [
    'connections' => [
        'mongodb' => [
            'driver' => 'mongodb',
            'host' => env('MONGODB_HOST', '127.0.0.1'),
            'port' => env('MONGODB_PORT', 27017),
            'database' => env('MONGODB_DATABASE', 'myapp'),
            'username' => env('MONGODB_USERNAME', ''),
            'password' => env('MONGODB_PASSWORD', ''),
            'options' => [
                'database' => env('MONGODB_AUTH_DATABASE', 'admin'),
            ],
        ],
    ],
];
```

### Modèle Laravel avec MongoDB

```php
<?php
// app/Models/User.php

namespace App\Models;

use MongoDB\Laravel\Eloquent\Model;

class User extends Model
{
    protected $connection = 'mongodb';
    protected $collection = 'users';

    protected $fillable = [
        'name',
        'email',
        'age',
        'status',
        'roles',
        'preferences',
    ];

    protected $casts = [
        'age' => 'integer',
        'roles' => 'array',
        'preferences' => 'array',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
        'last_login' => 'datetime',
        'login_count' => 'integer',
    ];

    protected $attributes = [
        'status' => 'active',
        'roles' => ['user'],
        'login_count' => 0,
    ];

    // Relations
    public function orders()
    {
        return $this->hasMany(Order::class);
    }

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }

    // Mutateurs
    public function setPasswordAttribute($value)
    {
        $this->attributes['password'] = bcrypt($value);
    }
}
```

### Controller Laravel

```php
<?php
// app/Http/Controllers/UserController.php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class UserController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        $users = User::active()
            ->orderBy('created_at', 'desc')
            ->paginate(20);

        return response()->json($users);
    }

    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => 'required|string|min:2|max:100',
            'email' => 'required|email|unique:users,email',
            'age' => 'nullable|integer|min:18|max:120',
            'password' => 'required|string|min:8',
        ]);

        $user = User::create($validated);

        return response()->json($user, 201);
    }

    public function show(string $id): JsonResponse
    {
        $user = User::findOrFail($id);
        return response()->json($user);
    }

    public function update(Request $request, string $id): JsonResponse
    {
        $user = User::findOrFail($id);

        $validated = $request->validate([
            'name' => 'sometimes|string|min:2|max:100',
            'email' => 'sometimes|email|unique:users,email,' . $id,
            'age' => 'sometimes|integer|min:18|max:120',
        ]);

        $user->update($validated);

        return response()->json($user);
    }

    public function destroy(string $id): JsonResponse
    {
        $user = User::findOrFail($id);
        $user->delete();

        return response()->json(['message' => 'User deleted']);
    }

    public function search(Request $request): JsonResponse
    {
        $query = $request->input('q');

        $users = User::where('name', 'regex', "/{$query}/i")
            ->orWhere('email', 'regex', "/{$query}/i")
            ->limit(20)
            ->get();

        return response()->json($users);
    }
}
```

## Intégration Symfony

### Configuration Symfony

```yaml
# config/packages/doctrine_mongodb.yaml
doctrine_mongodb:
    connections:
        default:
            server: '%env(MONGODB_URI)%'
            options: {}
    default_database: '%env(MONGODB_DATABASE)%'
    document_managers:
        default:
            auto_mapping: true
            mappings:
                App:
                    is_bundle: false
                    type: annotation
                    dir: '%kernel.project_dir%/src/Document'
                    prefix: 'App\Document'
                    alias: App
```

### Document Symfony

```php
<?php
// src/Document/User.php

namespace App\Document;

use Doctrine\ODM\MongoDB\Mapping\Annotations as MongoDB;
use Symfony\Component\Validator\Constraints as Assert;

#[MongoDB\Document(collection: 'users')]
#[MongoDB\Index(keys: ['email' => 'asc'], options: ['unique' => true])]
#[MongoDB\Index(keys: ['status' => 'asc'])]
class User
{
    #[MongoDB\Id]
    private ?string $id = null;

    #[MongoDB\Field(type: 'string')]
    #[Assert\NotBlank]
    #[Assert\Length(min: 2, max: 100)]
    private string $name;

    #[MongoDB\Field(type: 'string')]
    #[Assert\NotBlank]
    #[Assert\Email]
    private string $email;

    #[MongoDB\Field(type: 'int')]
    #[Assert\Range(min: 18, max: 120)]
    private ?int $age = null;

    #[MongoDB\Field(type: 'string')]
    private string $status = 'active';

    #[MongoDB\Field(type: 'collection')]
    private array $roles = ['ROLE_USER'];

    #[MongoDB\Field(type: 'date')]
    private \DateTime $createdAt;

    #[MongoDB\Field(type: 'date')]
    private \DateTime $updatedAt;

    #[MongoDB\Field(type: 'int')]
    private int $loginCount = 0;

    public function __construct()
    {
        $this->createdAt = new \DateTime();
        $this->updatedAt = new \DateTime();
    }

    // Getters and setters...

    public function getId(): ?string
    {
        return $this->id;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): self
    {
        $this->name = $name;
        return $this;
    }

    // ... autres getters/setters
}
```

### Repository Symfony

```php
<?php
// src/Repository/UserRepository.php

namespace App\Repository;

use App\Document\User;
use Doctrine\ODM\MongoDB\Repository\DocumentRepository;

class UserRepository extends DocumentRepository
{
    public function findByEmail(string $email): ?User
    {
        return $this->findOneBy(['email' => $email]);
    }

    public function findActiveUsers(int $limit = 100): array
    {
        return $this->createQueryBuilder()
            ->field('status')->equals('active')
            ->sort('createdAt', 'DESC')
            ->limit($limit)
            ->getQuery()
            ->execute()
            ->toArray();
    }

    public function searchUsers(string $term): array
    {
        return $this->createQueryBuilder()
            ->addOr($this->createQueryBuilder()->expr()->field('name')->equals(new \MongoDB\BSON\Regex($term, 'i')))
            ->addOr($this->createQueryBuilder()->expr()->field('email')->equals(new \MongoDB\BSON\Regex($term, 'i')))
            ->getQuery()
            ->execute()
            ->toArray();
    }
}
```

## Gestion d'erreurs

```php
<?php
// src/Exception/MongoDBException.php

namespace App\Exception;

class MongoDBException extends \RuntimeException
{
    public static function fromDriverException(\Exception $e): self
    {
        if ($e instanceof \MongoDB\Driver\Exception\BulkWriteException) {
            if ($e->getCode() === 11000) {
                return new self('Duplicate key error', 409, $e);
            }
        }

        if ($e instanceof \MongoDB\Driver\Exception\ConnectionTimeoutException) {
            return new self('Connection timeout', 503, $e);
        }

        if ($e instanceof \MongoDB\Driver\Exception\AuthenticationException) {
            return new self('Authentication failed', 401, $e);
        }

        return new self('Database error: ' . $e->getMessage(), 500, $e);
    }
}
```

## Bonnes pratiques

### ✅ DO - À faire

```php
// 1. Utiliser les typed properties (PHP 8.0+)
public function __construct(
    public string $name,
    public string $email,
) {}

// 2. Utiliser readonly (PHP 8.1+) pour DTOs
readonly class CreateUserDTO {}

// 3. Utiliser les enums (PHP 8.1+)
enum UserStatus: string {
    case ACTIVE = 'active';
}

// 4. Typer les retours
public function findById(string $id): ?User

// 5. Utiliser try-catch pour les exceptions MongoDB
try {
    $collection->insertOne($document);
} catch (\MongoDB\Driver\Exception\BulkWriteException $e) {
    // Gérer erreur duplicate key
}

// 6. Fermer les curseurs
$cursor = $collection->find($filter);
// Utiliser iterator_to_array() ou foreach

// 7. Utiliser UTCDateTime pour les dates
new \MongoDB\BSON\UTCDateTime()

// 8. Valider les ObjectId
if (!ObjectId::isValid($id)) {
    throw new \InvalidArgumentException('Invalid ID');
}

// 9. Logger les opérations importantes
$logger->info('User created', ['id' => $user->id]);

// 10. Utiliser les indexes
$collection->createIndex(['email' => 1], ['unique' => true]);
```

### ❌ DON'T - À éviter

```php
// ❌ Créer un nouveau client par requête
// ❌ Ne pas typer les propriétés et retours
// ❌ Ignorer les exceptions
// ❌ Ne pas valider les ObjectId
// ❌ Utiliser DateTime au lieu de UTCDateTime
// ❌ Ne pas utiliser d'index
// ❌ Exposer les données sensibles dans les APIs
// ❌ Ne pas fermer les connexions/curseurs
// ❌ Faire des requêtes N+1
```

## Comparaison des approches

| Approche | Avantages | Inconvénients |
|----------|-----------|---------------|
| **Native** | Contrôle total, Flexible | Plus verbeux |
| **Laravel Eloquent** | Simple, Familier | Moins performant pour cas complexes |
| **Symfony Doctrine ODM** | Type-safe, Annotations | Courbe d'apprentissage |

## Conclusion

Le driver PHP MongoDB offre une excellente intégration avec l'écosystème PHP :

1. **Extension C** : Hautes performances
2. **Library PHP** : API haut niveau idiomatique
3. **PHP 8.x** : Typed properties, enums, readonly
4. **Frameworks** : Laravel, Symfony support
5. **ODM** : Abstraction pour simplifier le code

**Points clés** :
- Extension C requise (mongodb)
- Library PHP pour API haut niveau
- Singleton pour le client
- Typed properties PHP 8+
- Enums pour statuts
- UTCDateTime pour dates
- Validation des ObjectId
- Gestion d'erreurs robuste

---


⏭️ [Driver Ruby](/15-drivers-integration-applicative/08-driver-ruby.md)
