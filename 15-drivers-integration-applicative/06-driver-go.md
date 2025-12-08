🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.6 Driver Go

## Introduction

Le driver Go MongoDB est un driver officiel moderne qui s'intègre parfaitement avec les idiomes Go. Il offre une excellente performance grâce à la nature compilée de Go, un support natif des goroutines pour la concurrence, et une API claire et typée. Cette section couvre l'utilisation du driver avec **Go 1.21+** et les meilleures pratiques pour des applications de production.

## Installation et configuration

### Installation

```bash
# Initialiser un module Go
go mod init myapp

# Installer le driver MongoDB
go get go.mongodb.org/mongo-driver/mongo
go get go.mongodb.org/mongo-driver/bson

# Version minimale recommandée : 1.13+
# Go version recommandée : 1.21+
```

### Structure de projet

```
myapp/
├── go.mod
├── go.sum
├── main.go
├── config/
│   └── database.go
├── models/
│   ├── user.go
│   └── types.go
├── repository/
│   ├── repository.go
│   └── user_repository.go
├── service/
│   └── user_service.go
├── handler/
│   └── user_handler.go
└── internal/
    ├── logger/
    │   └── logger.go
    └── errors/
        └── errors.go
```

### go.mod

```go
module myapp

go 1.21

require (
    go.mongodb.org/mongo-driver v1.13.1
    github.com/joho/godotenv v1.5.1
    go.uber.org/zap v1.26.0
)

require (
    github.com/golang/snappy v0.0.4 // indirect
    github.com/klauspost/compress v1.17.0 // indirect
    github.com/montanaflynn/stats v0.7.1 // indirect
    github.com/xdg-go/pbkdf2 v1.0.0 // indirect
    github.com/xdg-go/scram v1.1.2 // indirect
    github.com/xdg-go/stringprep v1.0.4 // indirect
    github.com/youmark/pkcs8 v0.0.0-20201027041543-1326539a0a0a // indirect
    golang.org/x/crypto v0.17.0 // indirect
    golang.org/x/sync v0.5.0 // indirect
    golang.org/x/text v0.14.0 // indirect
)
```

## Configuration

### Configuration avec environnement

```go
// config/database.go
package config

import (
    "context"
    "fmt"
    "os"
    "strconv"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.mongodb.org/mongo-driver/mongo/readpref"
    "go.uber.org/zap"
)

// DatabaseConfig contient la configuration MongoDB
type DatabaseConfig struct {
    URI                      string
    Database                 string
    MaxPoolSize              uint64
    MinPoolSize              uint64
    MaxConnIdleTime          time.Duration
    ServerSelectionTimeout   time.Duration
    SocketTimeout            time.Duration
    ConnectTimeout           time.Duration
}

// LoadConfig charge la configuration depuis l'environnement
func LoadConfig() *DatabaseConfig {
    return &DatabaseConfig{
        URI:                    getEnv("MONGODB_URI", "mongodb://localhost:27017"),
        Database:               getEnv("MONGODB_DATABASE", "myapp"),
        MaxPoolSize:            getEnvUint64("MONGODB_MAX_POOL_SIZE", 50),
        MinPoolSize:            getEnvUint64("MONGODB_MIN_POOL_SIZE", 10),
        MaxConnIdleTime:        getEnvDuration("MONGODB_MAX_CONN_IDLE_TIME", 30*time.Second),
        ServerSelectionTimeout: getEnvDuration("MONGODB_SERVER_SELECTION_TIMEOUT", 5*time.Second),
        SocketTimeout:          getEnvDuration("MONGODB_SOCKET_TIMEOUT", 45*time.Second),
        ConnectTimeout:         getEnvDuration("MONGODB_CONNECT_TIMEOUT", 10*time.Second),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvUint64(key string, defaultValue uint64) uint64 {
    if value := os.Getenv(key); value != "" {
        if parsed, err := strconv.ParseUint(value, 10, 64); err == nil {
            return parsed
        }
    }
    return defaultValue
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if parsed, err := time.ParseDuration(value); err == nil {
            return parsed
        }
    }
    return defaultValue
}

// Database représente une connexion MongoDB
type Database struct {
    client *mongo.Client
    db     *mongo.Database
    config *DatabaseConfig
    logger *zap.Logger
}

// NewDatabase crée une nouvelle connexion MongoDB
func NewDatabase(config *DatabaseConfig, logger *zap.Logger) (*Database, error) {
    ctx, cancel := context.WithTimeout(context.Background(), config.ConnectTimeout)
    defer cancel()

    // Options du client
    clientOptions := options.Client().
        ApplyURI(config.URI).
        SetMaxPoolSize(config.MaxPoolSize).
        SetMinPoolSize(config.MinPoolSize).
        SetMaxConnIdleTime(config.MaxConnIdleTime).
        SetServerSelectionTimeout(config.ServerSelectionTimeout).
        SetSocketTimeout(config.SocketTimeout).
        SetConnectTimeout(config.ConnectTimeout).
        SetRetryWrites(true).
        SetRetryReads(true)

    // Monitoring (optionnel)
    monitor := &event.CommandMonitor{
        Started: func(ctx context.Context, e *event.CommandStartedEvent) {
            logger.Debug("MongoDB command started",
                zap.String("command", e.CommandName),
                zap.Int64("requestID", e.RequestID))
        },
        Succeeded: func(ctx context.Context, e *event.CommandSucceededEvent) {
            logger.Debug("MongoDB command succeeded",
                zap.String("command", e.CommandName),
                zap.Int64("duration", e.DurationNanos/1000000))
        },
        Failed: func(ctx context.Context, e *event.CommandFailedEvent) {
            logger.Error("MongoDB command failed",
                zap.String("command", e.CommandName),
                zap.String("error", e.Failure))
        },
    }
    clientOptions.SetMonitor(monitor)

    // Connexion
    client, err := mongo.Connect(ctx, clientOptions)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to MongoDB: %w", err)
    }

    // Ping pour vérifier la connexion
    if err := client.Ping(ctx, readpref.Primary()); err != nil {
        return nil, fmt.Errorf("failed to ping MongoDB: %w", err)
    }

    logger.Info("✅ Connected to MongoDB",
        zap.String("database", config.Database))

    return &Database{
        client: client,
        db:     client.Database(config.Database),
        config: config,
        logger: logger,
    }, nil
}

// GetClient retourne le client MongoDB
func (d *Database) GetClient() *mongo.Client {
    return d.client
}

// GetDatabase retourne la database MongoDB
func (d *Database) GetDatabase() *mongo.Database {
    return d.db
}

// Close ferme la connexion MongoDB
func (d *Database) Close(ctx context.Context) error {
    if err := d.client.Disconnect(ctx); err != nil {
        return fmt.Errorf("failed to disconnect from MongoDB: %w", err)
    }
    d.logger.Info("🔌 Disconnected from MongoDB")
    return nil
}
```

## Modèles de données

### Types de base

```go
// models/types.go
package models

import "time"

// UserStatus représente le statut d'un utilisateur
type UserStatus string

const (
    UserStatusActive    UserStatus = "active"
    UserStatusInactive  UserStatus = "inactive"
    UserStatusSuspended UserStatus = "suspended"
    UserStatusDeleted   UserStatus = "deleted"
)

// IsValid vérifie si le statut est valide
func (s UserStatus) IsValid() bool {
    switch s {
    case UserStatusActive, UserStatusInactive, UserStatusSuspended, UserStatusDeleted:
        return true
    default:
        return false
    }
}

// UserPreferences représente les préférences utilisateur
type UserPreferences struct {
    Language      string `bson:"language"`
    Timezone      string `bson:"timezone"`
    Notifications bool   `bson:"notifications"`
}

// DefaultPreferences retourne les préférences par défaut
func DefaultPreferences() UserPreferences {
    return UserPreferences{
        Language:      "en",
        Timezone:      "UTC",
        Notifications: true,
    }
}
```

### Modèle utilisateur

```go
// models/user.go
package models

import (
    "time"

    "go.mongodb.org/mongo-driver/bson/primitive"
)

// User représente un utilisateur
type User struct {
    ID          primitive.ObjectID `bson:"_id,omitempty"`
    Name        string             `bson:"name"`
    Email       string             `bson:"email"`
    Age         *int               `bson:"age,omitempty"`
    Status      UserStatus         `bson:"status"`
    Roles       []string           `bson:"roles"`
    Preferences *UserPreferences   `bson:"preferences,omitempty"`
    CreatedAt   time.Time          `bson:"created_at"`
    UpdatedAt   time.Time          `bson:"updated_at"`
    LastLogin   *time.Time         `bson:"last_login,omitempty"`
    LoginCount  int                `bson:"login_count"`
}

// NewUser crée un nouvel utilisateur avec des valeurs par défaut
func NewUser(name, email string) *User {
    now := time.Now()
    return &User{
        Name:       name,
        Email:      email,
        Status:     UserStatusActive,
        Roles:      []string{"user"},
        CreatedAt:  now,
        UpdatedAt:  now,
        LoginCount: 0,
    }
}

// IncrementLogin incrémente le compteur de connexion
func (u *User) IncrementLogin() {
    u.LoginCount++
    now := time.Now()
    u.LastLogin = &now
    u.UpdatedAt = now
}

// CreateUserInput représente les données de création
type CreateUserInput struct {
    Name     string   `json:"name" validate:"required,min=2,max=100"`
    Email    string   `json:"email" validate:"required,email"`
    Age      *int     `json:"age,omitempty" validate:"omitempty,min=18,max=120"`
    Password string   `json:"password" validate:"required,min=8"`
    Roles    []string `json:"roles,omitempty"`
}

// UpdateUserInput représente les données de mise à jour
type UpdateUserInput struct {
    Name  *string `json:"name,omitempty" validate:"omitempty,min=2,max=100"`
    Email *string `json:"email,omitempty" validate:"omitempty,email"`
    Age   *int    `json:"age,omitempty" validate:"omitempty,min=18,max=120"`
}

// UserPublic représente un utilisateur pour l'API publique
type UserPublic struct {
    ID        string     `json:"id"`
    Name      string     `json:"name"`
    Email     string     `json:"email"`
    Status    UserStatus `json:"status"`
    CreatedAt time.Time  `json:"created_at"`
}

// ToPublic convertit un User en UserPublic
func (u *User) ToPublic() *UserPublic {
    return &UserPublic{
        ID:        u.ID.Hex(),
        Name:      u.Name,
        Email:     u.Email,
        Status:    u.Status,
        CreatedAt: u.CreatedAt,
    }
}

// UserStatistics représente les statistiques utilisateurs
type UserStatistics struct {
    Total         int64            `json:"total"`
    ByStatus      map[string]int64 `json:"by_status"`
    RecentSignups int64            `json:"recent_signups"`
}
```

## Repository Pattern

### Interface de base

```go
// repository/repository.go
package repository

import (
    "context"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

// BaseRepository définit l'interface de base pour un repository
type BaseRepository interface {
    FindByID(ctx context.Context, id primitive.ObjectID) (interface{}, error)
    FindOne(ctx context.Context, filter bson.M) (interface{}, error)
    FindMany(ctx context.Context, filter bson.M, opts ...*options.FindOptions) ([]interface{}, error)
    InsertOne(ctx context.Context, document interface{}) (primitive.ObjectID, error)
    UpdateOne(ctx context.Context, filter bson.M, update bson.M) (bool, error)
    UpdateByID(ctx context.Context, id primitive.ObjectID, update bson.M) (bool, error)
    DeleteOne(ctx context.Context, filter bson.M) (bool, error)
    DeleteByID(ctx context.Context, id primitive.ObjectID) (bool, error)
    Count(ctx context.Context, filter bson.M) (int64, error)
    Exists(ctx context.Context, filter bson.M) (bool, error)
}
```

### Repository utilisateur

```go
// repository/user_repository.go
package repository

import (
    "context"
    "fmt"
    "time"

    "myapp/models"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.uber.org/zap"
)

// UserRepository gère les opérations sur les utilisateurs
type UserRepository struct {
    collection *mongo.Collection
    logger     *zap.Logger
}

// NewUserRepository crée un nouveau UserRepository
func NewUserRepository(db *mongo.Database, logger *zap.Logger) (*UserRepository, error) {
    collection := db.Collection("users")

    repo := &UserRepository{
        collection: collection,
        logger:     logger,
    }

    // Créer les index
    if err := repo.ensureIndexes(context.Background()); err != nil {
        return nil, fmt.Errorf("failed to ensure indexes: %w", err)
    }

    return repo, nil
}

// ensureIndexes crée les index nécessaires
func (r *UserRepository) ensureIndexes(ctx context.Context) error {
    indexes := []mongo.IndexModel{
        {
            Keys:    bson.D{{Key: "email", Value: 1}},
            Options: options.Index().SetUnique(true),
        },
        {
            Keys: bson.D{{Key: "status", Value: 1}},
        },
        {
            Keys: bson.D{
                {Key: "status", Value: 1},
                {Key: "created_at", Value: -1},
            },
        },
    }

    _, err := r.collection.Indexes().CreateMany(ctx, indexes)
    if err != nil {
        return err
    }

    r.logger.Info("Indexes created for users collection")
    return nil
}

// Create crée un nouvel utilisateur
func (r *UserRepository) Create(ctx context.Context, user *models.User) error {
    result, err := r.collection.InsertOne(ctx, user)
    if err != nil {
        if mongo.IsDuplicateKeyError(err) {
            return fmt.Errorf("email already exists: %w", err)
        }
        return fmt.Errorf("failed to insert user: %w", err)
    }

    user.ID = result.InsertedID.(primitive.ObjectID)
    return nil
}

// FindByID trouve un utilisateur par ID
func (r *UserRepository) FindByID(ctx context.Context, id primitive.ObjectID) (*models.User, error) {
    var user models.User

    err := r.collection.FindOne(ctx, bson.M{"_id": id}).Decode(&user)
    if err != nil {
        if err == mongo.ErrNoDocuments {
            return nil, nil
        }
        return nil, fmt.Errorf("failed to find user: %w", err)
    }

    return &user, nil
}

// FindByEmail trouve un utilisateur par email
func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*models.User, error) {
    var user models.User

    err := r.collection.FindOne(ctx, bson.M{"email": email}).Decode(&user)
    if err != nil {
        if err == mongo.ErrNoDocuments {
            return nil, nil
        }
        return nil, fmt.Errorf("failed to find user by email: %w", err)
    }

    return &user, nil
}

// FindActiveUsers trouve les utilisateurs actifs avec pagination
func (r *UserRepository) FindActiveUsers(ctx context.Context, limit, skip int64) ([]*models.User, error) {
    filter := bson.M{"status": models.UserStatusActive}

    opts := options.Find().
        SetLimit(limit).
        SetSkip(skip).
        SetSort(bson.D{{Key: "created_at", Value: -1}})

    cursor, err := r.collection.Find(ctx, filter, opts)
    if err != nil {
        return nil, fmt.Errorf("failed to find active users: %w", err)
    }
    defer cursor.Close(ctx)

    var users []*models.User
    if err := cursor.All(ctx, &users); err != nil {
        return nil, fmt.Errorf("failed to decode users: %w", err)
    }

    return users, nil
}

// Update met à jour un utilisateur
func (r *UserRepository) Update(ctx context.Context, id primitive.ObjectID, update bson.M) error {
    // Ajouter updated_at automatiquement
    if update["$set"] == nil {
        update["$set"] = bson.M{}
    }
    update["$set"].(bson.M)["updated_at"] = time.Now()

    result, err := r.collection.UpdateOne(
        ctx,
        bson.M{"_id": id},
        update,
    )
    if err != nil {
        return fmt.Errorf("failed to update user: %w", err)
    }

    if result.MatchedCount == 0 {
        return fmt.Errorf("user not found")
    }

    return nil
}

// IncrementLoginCount incrémente le compteur de connexion
func (r *UserRepository) IncrementLoginCount(ctx context.Context, id primitive.ObjectID) error {
    now := time.Now()
    update := bson.M{
        "$inc": bson.M{"login_count": 1},
        "$set": bson.M{
            "last_login":  now,
            "updated_at": now,
        },
    }

    _, err := r.collection.UpdateOne(ctx, bson.M{"_id": id}, update)
    if err != nil {
        return fmt.Errorf("failed to increment login count: %w", err)
    }

    return nil
}

// Search recherche des utilisateurs par nom ou email
func (r *UserRepository) Search(ctx context.Context, searchTerm string, limit int64) ([]*models.User, error) {
    filter := bson.M{
        "$or": []bson.M{
            {"name": bson.M{"$regex": searchTerm, "$options": "i"}},
            {"email": bson.M{"$regex": searchTerm, "$options": "i"}},
        },
    }

    opts := options.Find().SetLimit(limit)

    cursor, err := r.collection.Find(ctx, filter, opts)
    if err != nil {
        return nil, fmt.Errorf("failed to search users: %w", err)
    }
    defer cursor.Close(ctx)

    var users []*models.User
    if err := cursor.All(ctx, &users); err != nil {
        return nil, fmt.Errorf("failed to decode users: %w", err)
    }

    return users, nil
}

// FindByRole trouve les utilisateurs ayant un rôle spécifique
func (r *UserRepository) FindByRole(ctx context.Context, role string) ([]*models.User, error) {
    filter := bson.M{"roles": role}

    cursor, err := r.collection.Find(ctx, filter)
    if err != nil {
        return nil, fmt.Errorf("failed to find users by role: %w", err)
    }
    defer cursor.Close(ctx)

    var users []*models.User
    if err := cursor.All(ctx, &users); err != nil {
        return nil, fmt.Errorf("failed to decode users: %w", err)
    }

    return users, nil
}

// BulkUpdateStatus met à jour le statut de plusieurs utilisateurs
func (r *UserRepository) BulkUpdateStatus(ctx context.Context, userIDs []primitive.ObjectID, status models.UserStatus) (int64, error) {
    filter := bson.M{"_id": bson.M{"$in": userIDs}}
    update := bson.M{
        "$set": bson.M{
            "status":     status,
            "updated_at": time.Now(),
        },
    }

    result, err := r.collection.UpdateMany(ctx, filter, update)
    if err != nil {
        return 0, fmt.Errorf("failed to bulk update status: %w", err)
    }

    return result.ModifiedCount, nil
}

// Delete supprime un utilisateur
func (r *UserRepository) Delete(ctx context.Context, id primitive.ObjectID) error {
    result, err := r.collection.DeleteOne(ctx, bson.M{"_id": id})
    if err != nil {
        return fmt.Errorf("failed to delete user: %w", err)
    }

    if result.DeletedCount == 0 {
        return fmt.Errorf("user not found")
    }

    return nil
}

// Count compte les utilisateurs
func (r *UserRepository) Count(ctx context.Context, filter bson.M) (int64, error) {
    count, err := r.collection.CountDocuments(ctx, filter)
    if err != nil {
        return 0, fmt.Errorf("failed to count users: %w", err)
    }
    return count, nil
}

// GetStatistics récupère les statistiques utilisateurs
func (r *UserRepository) GetStatistics(ctx context.Context) (*models.UserStatistics, error) {
    // Total
    total, err := r.Count(ctx, bson.M{})
    if err != nil {
        return nil, err
    }

    // Par statut
    pipeline := mongo.Pipeline{
        {{Key: "$group", Value: bson.M{
            "_id":   "$status",
            "count": bson.M{"$sum": 1},
        }}},
    }

    cursor, err := r.collection.Aggregate(ctx, pipeline)
    if err != nil {
        return nil, fmt.Errorf("failed to aggregate by status: %w", err)
    }
    defer cursor.Close(ctx)

    byStatus := make(map[string]int64)
    for cursor.Next(ctx) {
        var result struct {
            ID    string `bson:"_id"`
            Count int64  `bson:"count"`
        }
        if err := cursor.Decode(&result); err != nil {
            return nil, fmt.Errorf("failed to decode status count: %w", err)
        }
        byStatus[result.ID] = result.Count
    }

    // Inscriptions récentes (30 derniers jours)
    thirtyDaysAgo := time.Now().AddDate(0, 0, -30)
    recentFilter := bson.M{"created_at": bson.M{"$gte": thirtyDaysAgo}}
    recentSignups, err := r.Count(ctx, recentFilter)
    if err != nil {
        return nil, err
    }

    return &models.UserStatistics{
        Total:         total,
        ByStatus:      byStatus,
        RecentSignups: recentSignups,
    }, nil
}
```

## Service Layer

```go
// service/user_service.go
package service

import (
    "context"
    "fmt"

    "myapp/models"
    "myapp/repository"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.uber.org/zap"
)

// UserService gère la logique métier des utilisateurs
type UserService struct {
    repo   *repository.UserRepository
    logger *zap.Logger
}

// NewUserService crée un nouveau UserService
func NewUserService(repo *repository.UserRepository, logger *zap.Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger,
    }
}

// RegisterUser enregistre un nouvel utilisateur
func (s *UserService) RegisterUser(ctx context.Context, input *models.CreateUserInput) (*models.UserPublic, error) {
    // Vérifier si l'email existe
    existing, err := s.repo.FindByEmail(ctx, input.Email)
    if err != nil {
        return nil, fmt.Errorf("failed to check email: %w", err)
    }
    if existing != nil {
        return nil, fmt.Errorf("email already registered")
    }

    // Créer l'utilisateur
    user := models.NewUser(input.Name, input.Email)
    user.Age = input.Age

    if len(input.Roles) > 0 {
        user.Roles = input.Roles
    }

    // TODO: Hasher le mot de passe
    // user.PasswordHash = hashPassword(input.Password)

    if err := s.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }

    s.logger.Info("New user registered",
        zap.String("email", user.Email),
        zap.String("id", user.ID.Hex()))

    return user.ToPublic(), nil
}

// AuthenticateUser authentifie un utilisateur
func (s *UserService) AuthenticateUser(ctx context.Context, email, password string) (*models.User, error) {
    user, err := s.repo.FindByEmail(ctx, email)
    if err != nil {
        return nil, fmt.Errorf("failed to find user: %w", err)
    }
    if user == nil {
        return nil, fmt.Errorf("invalid credentials")
    }

    if user.Status != models.UserStatusActive {
        return nil, fmt.Errorf("account is not active")
    }

    // TODO: Vérifier le mot de passe
    // if !verifyPassword(password, user.PasswordHash) {
    //     return nil, fmt.Errorf("invalid credentials")
    // }

    // Incrémenter le compteur de connexion
    if err := s.repo.IncrementLoginCount(ctx, user.ID); err != nil {
        s.logger.Error("Failed to increment login count",
            zap.Error(err),
            zap.String("userID", user.ID.Hex()))
    }

    s.logger.Info("User authenticated",
        zap.String("email", user.Email),
        zap.String("id", user.ID.Hex()))

    return user, nil
}

// GetUserByID récupère un utilisateur par ID
func (s *UserService) GetUserByID(ctx context.Context, id string) (*models.UserPublic, error) {
    objectID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return nil, fmt.Errorf("invalid user ID: %w", err)
    }

    user, err := s.repo.FindByID(ctx, objectID)
    if err != nil {
        return nil, fmt.Errorf("failed to find user: %w", err)
    }
    if user == nil {
        return nil, fmt.Errorf("user not found")
    }

    return user.ToPublic(), nil
}

// UpdateUser met à jour un utilisateur
func (s *UserService) UpdateUser(ctx context.Context, id string, input *models.UpdateUserInput) (*models.UserPublic, error) {
    objectID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return nil, fmt.Errorf("invalid user ID: %w", err)
    }

    update := bson.M{"$set": bson.M{}}

    if input.Name != nil {
        update["$set"].(bson.M)["name"] = *input.Name
    }
    if input.Email != nil {
        update["$set"].(bson.M)["email"] = *input.Email
    }
    if input.Age != nil {
        update["$set"].(bson.M)["age"] = *input.Age
    }

    if len(update["$set"].(bson.M)) == 0 {
        return nil, fmt.Errorf("no updates provided")
    }

    if err := s.repo.Update(ctx, objectID, update); err != nil {
        return nil, fmt.Errorf("failed to update user: %w", err)
    }

    return s.GetUserByID(ctx, id)
}

// SearchUsers recherche des utilisateurs
func (s *UserService) SearchUsers(ctx context.Context, query string, limit int64) ([]*models.UserPublic, error) {
    users, err := s.repo.Search(ctx, query, limit)
    if err != nil {
        return nil, fmt.Errorf("failed to search users: %w", err)
    }

    result := make([]*models.UserPublic, len(users))
    for i, user := range users {
        result[i] = user.ToPublic()
    }

    return result, nil
}

// GetDashboardStats récupère les statistiques du dashboard
func (s *UserService) GetDashboardStats(ctx context.Context) (*models.UserStatistics, error) {
    return s.repo.GetStatistics(ctx)
}

// SuspendUser suspend un utilisateur
func (s *UserService) SuspendUser(ctx context.Context, id string) error {
    objectID, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return fmt.Errorf("invalid user ID: %w", err)
    }

    update := bson.M{
        "$set": bson.M{
            "status": models.UserStatusSuspended,
        },
    }

    if err := s.repo.Update(ctx, objectID, update); err != nil {
        return fmt.Errorf("failed to suspend user: %w", err)
    }

    s.logger.Info("User suspended", zap.String("id", id))
    return nil
}
```

## Opérations avancées

### Pagination

```go
// PaginatedResult représente un résultat paginé
type PaginatedResult struct {
    Data       interface{} `json:"data"`
    Total      int64       `json:"total"`
    Page       int64       `json:"page"`
    PageSize   int64       `json:"page_size"`
    TotalPages int64       `json:"total_pages"`
}

// FindUsersPaginated trouve des utilisateurs avec pagination
func (r *UserRepository) FindUsersPaginated(
    ctx context.Context,
    filter bson.M,
    page, pageSize int64,
) (*PaginatedResult, error) {
    // Compter le total
    total, err := r.collection.CountDocuments(ctx, filter)
    if err != nil {
        return nil, fmt.Errorf("failed to count documents: %w", err)
    }

    // Calculer le skip
    skip := (page - 1) * pageSize

    // Trouver les documents
    opts := options.Find().
        SetLimit(pageSize).
        SetSkip(skip).
        SetSort(bson.D{{Key: "created_at", Value: -1}})

    cursor, err := r.collection.Find(ctx, filter, opts)
    if err != nil {
        return nil, fmt.Errorf("failed to find documents: %w", err)
    }
    defer cursor.Close(ctx)

    var users []*models.User
    if err := cursor.All(ctx, &users); err != nil {
        return nil, fmt.Errorf("failed to decode documents: %w", err)
    }

    totalPages := (total + pageSize - 1) / pageSize

    return &PaginatedResult{
        Data:       users,
        Total:      total,
        Page:       page,
        PageSize:   pageSize,
        TotalPages: totalPages,
    }, nil
}
```

### Bulk Operations

```go
// BulkUpdatePrices met à jour les prix en masse
func BulkUpdatePrices(ctx context.Context, collection *mongo.Collection, updates []PriceUpdate) (int64, error) {
    var models []mongo.WriteModel

    for _, update := range updates {
        objectID, err := primitive.ObjectIDFromHex(update.ProductID)
        if err != nil {
            continue
        }

        model := mongo.NewUpdateOneModel().
            SetFilter(bson.M{"_id": objectID}).
            SetUpdate(bson.M{
                "$set": bson.M{
                    "price.amount": update.NewPrice,
                    "updated_at":   time.Now(),
                },
            })

        models = append(models, model)
    }

    opts := options.BulkWrite().SetOrdered(false)
    result, err := collection.BulkWrite(ctx, models, opts)
    if err != nil {
        return 0, fmt.Errorf("bulk write failed: %w", err)
    }

    return result.ModifiedCount, nil
}

type PriceUpdate struct {
    ProductID string
    NewPrice  float64
}
```

## Agrégations

```go
// GetSalesAnalytics récupère les analyses de ventes
func GetSalesAnalytics(
    ctx context.Context,
    collection *mongo.Collection,
    startDate, endDate time.Time,
) ([]SalesCategory, error) {
    pipeline := mongo.Pipeline{
        // Filtrer par période
        {{Key: "$match", Value: bson.M{
            "created_at": bson.M{
                "$gte": startDate,
                "$lte": endDate,
            },
            "status": "completed",
        }}},

        // Dérouler les items
        {{Key: "$unwind", Value: "$items"}},

        // Jointure avec products
        {{Key: "$lookup", Value: bson.M{
            "from":         "products",
            "localField":   "items.product_id",
            "foreignField": "_id",
            "as":           "product_details",
        }}},

        {{Key: "$unwind", Value: "$product_details"}},

        // Grouper par catégorie
        {{Key: "$group", Value: bson.M{
            "_id": "$product_details.category",
            "total_sales": bson.M{
                "$sum": bson.M{
                    "$multiply": []interface{}{
                        "$items.quantity",
                        "$items.price",
                    },
                },
            },
            "total_items": bson.M{"$sum": "$items.quantity"},
            "avg_order_value": bson.M{"$avg": "$total_amount"},
        }}},

        // Trier
        {{Key: "$sort", Value: bson.D{{Key: "total_sales", Value: -1}}}},
    }

    cursor, err := collection.Aggregate(ctx, pipeline)
    if err != nil {
        return nil, fmt.Errorf("aggregation failed: %w", err)
    }
    defer cursor.Close(ctx)

    var results []SalesCategory
    if err := cursor.All(ctx, &results); err != nil {
        return nil, fmt.Errorf("failed to decode results: %w", err)
    }

    return results, nil
}

type SalesCategory struct {
    Category        string  `bson:"_id"`
    TotalSales      float64 `bson:"total_sales"`
    TotalItems      int64   `bson:"total_items"`
    AvgOrderValue   float64 `bson:"avg_order_value"`
}
```

```go
// GetProductDashboard récupère les statistiques produits avec facettes
func GetProductDashboard(ctx context.Context, collection *mongo.Collection) (*ProductDashboard, error) {
    pipeline := mongo.Pipeline{
        {{Key: "$facet", Value: bson.M{
            "general_stats": []bson.M{
                {
                    "$group": bson.M{
                        "_id":            nil,
                        "total_products": bson.M{"$sum": 1},
                        "avg_price":      bson.M{"$avg": "$price.amount"},
                        "total_inventory": bson.M{"$sum": "$inventory.quantity"},
                    },
                },
            },
            "by_category": []bson.M{
                {
                    "$group": bson.M{
                        "_id":       "$category",
                        "count":     bson.M{"$sum": 1},
                        "avg_price": bson.M{"$avg": "$price.amount"},
                    },
                },
                {"$sort": bson.M{"count": -1}},
            },
            "top_products": []bson.M{
                {"$sort": bson.M{"reviews.rating": -1}},
                {"$limit": 10},
                {
                    "$project": bson.M{
                        "name":   1,
                        "price":  "$price.amount",
                        "rating": "$reviews.rating",
                    },
                },
            },
        }}},
    }

    cursor, err := collection.Aggregate(ctx, pipeline)
    if err != nil {
        return nil, fmt.Errorf("aggregation failed: %w", err)
    }
    defer cursor.Close(ctx)

    var results []bson.M
    if err := cursor.All(ctx, &results); err != nil {
        return nil, fmt.Errorf("failed to decode results: %w", err)
    }

    if len(results) == 0 {
        return nil, fmt.Errorf("no results returned")
    }

    return &ProductDashboard{
        GeneralStats: results[0]["general_stats"],
        ByCategory:   results[0]["by_category"],
        TopProducts:  results[0]["top_products"],
    }, nil
}

type ProductDashboard struct {
    GeneralStats interface{} `json:"general_stats"`
    ByCategory   interface{} `json:"by_category"`
    TopProducts  interface{} `json:"top_products"`
}
```

## Transactions

```go
// TransferFunds effectue un transfert de fonds avec transaction
func TransferFunds(
    ctx context.Context,
    client *mongo.Client,
    db *mongo.Database,
    fromAccountID, toAccountID primitive.ObjectID,
    amount float64,
) error {
    session, err := client.StartSession()
    if err != nil {
        return fmt.Errorf("failed to start session: %w", err)
    }
    defer session.EndSession(ctx)

    callback := func(sessCtx mongo.SessionContext) (interface{}, error) {
        accounts := db.Collection("accounts")

        // Débiter le compte source
        debitFilter := bson.M{
            "_id":     fromAccountID,
            "balance": bson.M{"$gte": amount},
        }
        debitUpdate := bson.M{
            "$inc": bson.M{"balance": -amount},
            "$push": bson.M{
                "transactions": bson.M{
                    "type":      "debit",
                    "amount":    amount,
                    "timestamp": time.Now(),
                },
            },
        }

        result, err := accounts.UpdateOne(sessCtx, debitFilter, debitUpdate)
        if err != nil {
            return nil, fmt.Errorf("debit failed: %w", err)
        }
        if result.MatchedCount == 0 {
            return nil, fmt.Errorf("insufficient funds or account not found")
        }

        // Créditer le compte destination
        creditFilter := bson.M{"_id": toAccountID}
        creditUpdate := bson.M{
            "$inc": bson.M{"balance": amount},
            "$push": bson.M{
                "transactions": bson.M{
                    "type":      "credit",
                    "amount":    amount,
                    "timestamp": time.Now(),
                },
            },
        }

        if _, err := accounts.UpdateOne(sessCtx, creditFilter, creditUpdate); err != nil {
            return nil, fmt.Errorf("credit failed: %w", err)
        }

        return nil, nil
    }

    _, err = session.WithTransaction(ctx, callback)
    if err != nil {
        return fmt.Errorf("transaction failed: %w", err)
    }

    return nil
}
```

```go
// CreateOrderWithRetry crée une commande avec retry automatique
func CreateOrderWithRetry(
    ctx context.Context,
    client *mongo.Client,
    db *mongo.Database,
    orderData *OrderData,
    maxRetries int,
) (primitive.ObjectID, error) {
    var lastErr error

    for attempt := 1; attempt <= maxRetries; attempt++ {
        orderID, err := createOrder(ctx, client, db, orderData)
        if err == nil {
            return orderID, nil
        }

        lastErr = err

        // Retry sur les erreurs transitoires
        if isTransientError(err) && attempt < maxRetries {
            time.Sleep(time.Duration(attempt*100) * time.Millisecond)
            continue
        }

        return primitive.NilObjectID, err
    }

    return primitive.NilObjectID, fmt.Errorf("failed after %d attempts: %w", maxRetries, lastErr)
}

func createOrder(
    ctx context.Context,
    client *mongo.Client,
    db *mongo.Database,
    orderData *OrderData,
) (primitive.ObjectID, error) {
    session, err := client.StartSession()
    if err != nil {
        return primitive.NilObjectID, fmt.Errorf("failed to start session: %w", err)
    }
    defer session.EndSession(ctx)

    var orderID primitive.ObjectID

    callback := func(sessCtx mongo.SessionContext) (interface{}, error) {
        orders := db.Collection("orders")
        products := db.Collection("products")
        users := db.Collection("users")

        // 1. Réserver l'inventaire
        for _, item := range orderData.Items {
            filter := bson.M{
                "_id":                item.ProductID,
                "inventory.quantity": bson.M{"$gte": item.Quantity},
            }
            update := bson.M{
                "$inc": bson.M{"inventory.quantity": -item.Quantity},
                "$set": bson.M{"updated_at": time.Now()},
            }

            result, err := products.UpdateOne(sessCtx, filter, update)
            if err != nil {
                return nil, fmt.Errorf("inventory update failed: %w", err)
            }
            if result.MatchedCount == 0 {
                return nil, fmt.Errorf("product %s out of stock", item.ProductID.Hex())
            }
        }

        // 2. Créer la commande
        order := bson.M{
            "user_id":      orderData.UserID,
            "items":        orderData.Items,
            "status":       "pending",
            "total_amount": orderData.TotalAmount,
            "created_at":   time.Now(),
            "updated_at":   time.Now(),
        }

        insertResult, err := orders.InsertOne(sessCtx, order)
        if err != nil {
            return nil, fmt.Errorf("order creation failed: %w", err)
        }

        orderID = insertResult.InsertedID.(primitive.ObjectID)

        // 3. Mettre à jour les stats utilisateur
        userUpdate := bson.M{
            "$inc": bson.M{"stats.total_orders": 1},
            "$set": bson.M{"stats.last_order_date": time.Now()},
        }

        if _, err := users.UpdateOne(sessCtx, bson.M{"_id": orderData.UserID}, userUpdate); err != nil {
            return nil, fmt.Errorf("user stats update failed: %w", err)
        }

        return nil, nil
    }

    _, err = session.WithTransaction(ctx, callback)
    if err != nil {
        return primitive.NilObjectID, err
    }

    return orderID, nil
}

func isTransientError(err error) bool {
    // Vérifier les erreurs transitoires MongoDB
    if cmdErr, ok := err.(mongo.CommandError); ok {
        return cmdErr.HasErrorLabel("TransientTransactionError")
    }
    return false
}

type OrderData struct {
    UserID      primitive.ObjectID
    Items       []OrderItem
    TotalAmount float64
}

type OrderItem struct {
    ProductID primitive.ObjectID `bson:"product_id"`
    Quantity  int                `bson:"quantity"`
    Price     float64            `bson:"price"`
}
```

## Concurrence avec Goroutines

```go
// ProcessUsersInParallel traite des utilisateurs en parallèle
func ProcessUsersInParallel(ctx context.Context, users []*models.User, workerCount int) error {
    // Canal pour distribuer le travail
    jobs := make(chan *models.User, len(users))
    // Canal pour les erreurs
    errors := make(chan error, len(users))

    // Démarrer les workers
    for w := 0; w < workerCount; w++ {
        go func() {
            for user := range jobs {
                if err := processUser(ctx, user); err != nil {
                    errors <- err
                } else {
                    errors <- nil
                }
            }
        }()
    }

    // Envoyer les jobs
    for _, user := range users {
        jobs <- user
    }
    close(jobs)

    // Collecter les résultats
    var errs []error
    for range users {
        if err := <-errors; err != nil {
            errs = append(errs, err)
        }
    }

    if len(errs) > 0 {
        return fmt.Errorf("processing failed for %d users", len(errs))
    }

    return nil
}

func processUser(ctx context.Context, user *models.User) error {
    // Traitement de l'utilisateur
    return nil
}
```

## Handler HTTP (avec Gin)

```go
// handler/user_handler.go
package handler

import (
    "net/http"

    "myapp/models"
    "myapp/service"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"
)

// UserHandler gère les endpoints utilisateurs
type UserHandler struct {
    service *service.UserService
    logger  *zap.Logger
}

// NewUserHandler crée un nouveau UserHandler
func NewUserHandler(service *service.UserService, logger *zap.Logger) *UserHandler {
    return &UserHandler{
        service: service,
        logger:  logger,
    }
}

// CreateUser crée un nouvel utilisateur
func (h *UserHandler) CreateUser(c *gin.Context) {
    var input models.CreateUserInput

    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.service.RegisterUser(c.Request.Context(), &input)
    if err != nil {
        h.logger.Error("Failed to register user", zap.Error(err))
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, user)
}

// GetUser récupère un utilisateur
func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")

    user, err := h.service.GetUserByID(c.Request.Context(), id)
    if err != nil {
        h.logger.Error("Failed to get user", zap.Error(err), zap.String("id", id))
        c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
        return
    }

    c.JSON(http.StatusOK, user)
}

// UpdateUser met à jour un utilisateur
func (h *UserHandler) UpdateUser(c *gin.Context) {
    id := c.Param("id")

    var input models.UpdateUserInput
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.service.UpdateUser(c.Request.Context(), id, &input)
    if err != nil {
        h.logger.Error("Failed to update user", zap.Error(err), zap.String("id", id))
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, user)
}

// SearchUsers recherche des utilisateurs
func (h *UserHandler) SearchUsers(c *gin.Context) {
    query := c.Query("q")

    users, err := h.service.SearchUsers(c.Request.Context(), query, 20)
    if err != nil {
        h.logger.Error("Failed to search users", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal server error"})
        return
    }

    c.JSON(http.StatusOK, users)
}

// GetStatistics récupère les statistiques
func (h *UserHandler) GetStatistics(c *gin.Context) {
    stats, err := h.service.GetDashboardStats(c.Request.Context())
    if err != nil {
        h.logger.Error("Failed to get statistics", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal server error"})
        return
    }

    c.JSON(http.StatusOK, stats)
}
```

## Gestion d'erreurs idiomatique

```go
// internal/errors/errors.go
package errors

import (
    "errors"
    "fmt"
)

// Erreurs prédéfinies
var (
    ErrNotFound          = errors.New("not found")
    ErrAlreadyExists     = errors.New("already exists")
    ErrInvalidInput      = errors.New("invalid input")
    ErrUnauthorized      = errors.New("unauthorized")
    ErrInternalServer    = errors.New("internal server error")
)

// AppError représente une erreur applicative
type AppError struct {
    Code    string
    Message string
    Err     error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// NewAppError crée une nouvelle AppError
func NewAppError(code, message string, err error) *AppError {
    return &AppError{
        Code:    code,
        Message: message,
        Err:     err,
    }
}
```

## Main Application

```go
// main.go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "myapp/config"
    "myapp/handler"
    "myapp/repository"
    "myapp/service"

    "github.com/gin-gonic/gin"
    "github.com/joho/godotenv"
    "go.uber.org/zap"
)

func main() {
    // Charger .env
    if err := godotenv.Load(); err != nil {
        log.Println("No .env file found")
    }

    // Logger
    logger, err := zap.NewProduction()
    if err != nil {
        log.Fatal("Failed to create logger:", err)
    }
    defer logger.Sync()

    // Configuration
    cfg := config.LoadConfig()

    // Database
    db, err := config.NewDatabase(cfg, logger)
    if err != nil {
        logger.Fatal("Failed to connect to database", zap.Error(err))
    }

    // Repositories
    userRepo, err := repository.NewUserRepository(db.GetDatabase(), logger)
    if err != nil {
        logger.Fatal("Failed to create user repository", zap.Error(err))
    }

    // Services
    userService := service.NewUserService(userRepo, logger)

    // Handlers
    userHandler := handler.NewUserHandler(userService, logger)

    // Router
    r := gin.Default()

    api := r.Group("/api")
    {
        users := api.Group("/users")
        {
            users.POST("", userHandler.CreateUser)
            users.GET("/:id", userHandler.GetUser)
            users.PUT("/:id", userHandler.UpdateUser)
            users.GET("/search", userHandler.SearchUsers)
            users.GET("/stats", userHandler.GetStatistics)
        }
    }

    // Graceful shutdown
    srv := &http.Server{
        Addr:    ":8080",
        Handler: r,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Fatal("Failed to start server", zap.Error(err))
        }
    }()

    logger.Info("Server started on :8080")

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    logger.Info("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        logger.Fatal("Server forced to shutdown", zap.Error(err))
    }

    if err := db.Close(ctx); err != nil {
        logger.Error("Failed to close database", zap.Error(err))
    }

    logger.Info("Server exited")
}
```

## Bonnes pratiques

### ✅ DO - À faire

```go
// 1. Toujours utiliser context
func FindUser(ctx context.Context, id string) (*User, error)

// 2. Error wrapping avec fmt.Errorf
return fmt.Errorf("failed to find user: %w", err)

// 3. Vérifier les erreurs immédiatement
if err != nil {
    return nil, err
}

// 4. Utiliser defer pour cleanup
defer cursor.Close(ctx)

// 5. Interfaces minimales
type UserRepository interface {
    FindByID(ctx context.Context, id primitive.ObjectID) (*User, error)
}

// 6. Structured logging
logger.Info("User created", zap.String("id", user.ID.Hex()))

// 7. Nil checks pour pointeurs
if user.Age != nil {
    // Utiliser *user.Age
}

// 8. Timeouts avec context
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 9. Graceful shutdown
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

// 10. Exports clairs
// User est exporté, user ne l'est pas
```

### ❌ DON'T - À éviter

```go
// ❌ Ignorer les erreurs
result, _ := collection.FindOne(ctx, filter)

// ❌ Ne pas utiliser context
func FindUser(id string) (*User, error)

// ❌ Panic dans le code business
panic("user not found")

// ❌ Bloquer sans timeout
collection.FindOne(context.Background(), filter)

// ❌ Ne pas fermer les curseurs
cursor, _ := collection.Find(ctx, filter)
// Manque defer cursor.Close(ctx)

// ❌ Créer un nouveau client par requête
// ❌ Ne pas vérifier ErrNoDocuments
// ❌ Mutations globales
// ❌ Ne pas logger les erreurs importantes
```

## Conclusion

Le driver Go MongoDB offre une excellente expérience pour les développeurs Go :

1. **Performance** : Compilation native, faible overhead
2. **Concurrence** : Goroutines pour traitement parallèle
3. **Type Safety** : Structs typées avec BSON tags
4. **Idiomatique** : Suit les conventions Go
5. **Context** : Support natif de context.Context
6. **Errors** : Gestion d'erreurs avec wrapping

**Points clés** :
- Context partout
- Error wrapping avec %w
- Defer pour cleanup
- Structured logging
- Interfaces minimales
- Graceful shutdown
- Goroutines pour concurrence

---


⏭️ [Driver PHP](/15-drivers-integration-applicative/07-driver-php.md)
