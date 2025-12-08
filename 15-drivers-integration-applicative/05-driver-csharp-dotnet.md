🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.5 Driver C# / .NET

## Introduction

Le driver C# MongoDB est un driver officiel mature et performant qui s'intègre parfaitement avec l'écosystème .NET. Il offre une API fortement typée, un excellent support async/await, et une intégration LINQ puissante. Cette section couvre l'utilisation moderne du driver avec **C# 12**, **.NET 8**, et les meilleures pratiques pour des applications de production.

## Installation et configuration

### Installation via NuGet

```bash
# .NET CLI
dotnet add package MongoDB.Driver
dotnet add package MongoDB.Bson

# Package Manager Console
Install-Package MongoDB.Driver
Install-Package MongoDB.Bson

# Pour ASP.NET Core
dotnet add package Microsoft.Extensions.Options.ConfigurationExtensions
```

### Structure de projet

```
MyApp/
├── MyApp.csproj
├── appsettings.json
├── appsettings.Development.json
├── Program.cs
├── Models/
│   ├── User.cs
│   ├── UserStatus.cs
│   └── UserPreferences.cs
├── DTOs/
│   ├── CreateUserDto.cs
│   ├── UpdateUserDto.cs
│   └── UserPublicDto.cs
├── Repositories/
│   ├── IBaseRepository.cs
│   ├── BaseRepository.cs
│   ├── IUserRepository.cs
│   └── UserRepository.cs
├── Services/
│   ├── IUserService.cs
│   └── UserService.cs
└── Configuration/
    ├── MongoDbSettings.cs
    └── MongoDbConfiguration.cs
```

### Configuration csproj

```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>12.0</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="MongoDB.Driver" Version="2.25.0" />
    <PackageReference Include="MongoDB.Bson" Version="2.25.0" />
    <PackageReference Include="Microsoft.Extensions.Options" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="8.0.0" />
    <PackageReference Include="FluentValidation.AspNetCore" Version="11.3.0" />
    <PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
  </ItemGroup>

</Project>
```

## Configuration

### appsettings.json

```json
{
  "MongoDbSettings": {
    "ConnectionString": "mongodb://localhost:27017",
    "DatabaseName": "myapp",
    "MaxConnectionPoolSize": 100,
    "MinConnectionPoolSize": 10,
    "ServerSelectionTimeout": "00:00:05",
    "SocketTimeout": "00:00:45",
    "ConnectTimeout": "00:00:10",
    "MaxConnectionIdleTime": "00:01:00"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MongoDB": "Debug"
    }
  }
}
```

### Classe de configuration

```csharp
// Configuration/MongoDbSettings.cs
namespace MyApp.Configuration;

public class MongoDbSettings
{
    public const string SectionName = "MongoDbSettings";

    public string ConnectionString { get; set; } = "mongodb://localhost:27017";
    public string DatabaseName { get; set; } = "myapp";
    public int MaxConnectionPoolSize { get; set; } = 100;
    public int MinConnectionPoolSize { get; set; } = 10;
    public TimeSpan ServerSelectionTimeout { get; set; } = TimeSpan.FromSeconds(5);
    public TimeSpan SocketTimeout { get; set; } = TimeSpan.FromSeconds(45);
    public TimeSpan ConnectTimeout { get; set; } = TimeSpan.FromSeconds(10);
    public TimeSpan MaxConnectionIdleTime { get; set; } = TimeSpan.FromMinutes(1);
}
```

```csharp
// Configuration/MongoDbConfiguration.cs
using MongoDB.Driver;
using MongoDB.Driver.Core.Configuration;
using Microsoft.Extensions.Options;

namespace MyApp.Configuration;

public class MongoDbConfiguration
{
    private readonly MongoDbSettings _settings;
    private readonly ILogger<MongoDbConfiguration> _logger;

    public MongoDbConfiguration(
        IOptions<MongoDbSettings> settings,
        ILogger<MongoDbConfiguration> logger)
    {
        _settings = settings.Value;
        _logger = logger;
    }

    public IMongoClient CreateClient()
    {
        var clientSettings = MongoClientSettings.FromConnectionString(_settings.ConnectionString);

        // Configuration du pool de connexions
        clientSettings.MaxConnectionPoolSize = _settings.MaxConnectionPoolSize;
        clientSettings.MinConnectionPoolSize = _settings.MinConnectionPoolSize;
        clientSettings.MaxConnectionIdleTime = _settings.MaxConnectionIdleTime;

        // Timeouts
        clientSettings.ServerSelectionTimeout = _settings.ServerSelectionTimeout;
        clientSettings.SocketTimeout = _settings.SocketTimeout;
        clientSettings.ConnectTimeout = _settings.ConnectTimeout;

        // Retry
        clientSettings.RetryWrites = true;
        clientSettings.RetryReads = true;

        // Logging
        clientSettings.ClusterConfigurator = builder =>
        {
            builder.Subscribe<MongoDB.Driver.Core.Events.CommandStartedEvent>(e =>
            {
                _logger.LogDebug("MongoDB Command Started: {CommandName}", e.CommandName);
            });

            builder.Subscribe<MongoDB.Driver.Core.Events.CommandSucceededEvent>(e =>
            {
                _logger.LogDebug("MongoDB Command Succeeded: {CommandName} ({Duration}ms)",
                    e.CommandName, e.Duration.TotalMilliseconds);
            });

            builder.Subscribe<MongoDB.Driver.Core.Events.CommandFailedEvent>(e =>
            {
                _logger.LogError("MongoDB Command Failed: {CommandName} - {Failure}",
                    e.CommandName, e.Failure);
            });
        };

        var client = new MongoClient(clientSettings);

        // Vérifier la connexion
        try
        {
            client.GetDatabase("admin").RunCommand<MongoDB.Bson.BsonDocument>(
                new MongoDB.Bson.BsonDocument("ping", 1)
            );
            _logger.LogInformation("✅ Connecté à MongoDB");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "❌ Échec de connexion MongoDB");
            throw;
        }

        return client;
    }
}
```

### Enregistrement dans Program.cs

```csharp
// Program.cs
using MyApp.Configuration;
using MyApp.Repositories;
using MyApp.Services;
using MongoDB.Driver;

var builder = WebApplication.CreateBuilder(args);

// Configuration MongoDB
builder.Services.Configure<MongoDbSettings>(
    builder.Configuration.GetSection(MongoDbSettings.SectionName)
);

// MongoDB Client (Singleton)
builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var config = new MongoDbConfiguration(
        sp.GetRequiredService<IOptions<MongoDbSettings>>(),
        sp.GetRequiredService<ILogger<MongoDbConfiguration>>()
    );
    return config.CreateClient();
});

// MongoDB Database (Singleton)
builder.Services.AddSingleton(sp =>
{
    var client = sp.GetRequiredService<IMongoClient>();
    var settings = sp.GetRequiredService<IOptions<MongoDbSettings>>().Value;
    return client.GetDatabase(settings.DatabaseName);
});

// Repositories
builder.Services.AddScoped<IUserRepository, UserRepository>();

// Services
builder.Services.AddScoped<IUserService, UserService>();

// Controllers, etc.
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Middleware pipeline...
app.MapControllers();
app.Run();
```

## Modèles de données

### Modèle utilisateur avec record

```csharp
// Models/User.cs
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;
using System.ComponentModel.DataAnnotations;

namespace MyApp.Models;

public record User
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string? Id { get; init; }

    [BsonElement("name")]
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public required string Name { get; init; }

    [BsonElement("email")]
    [Required]
    [EmailAddress]
    public required string Email { get; init; }

    [BsonElement("age")]
    [Range(18, 120)]
    public int? Age { get; init; }

    [BsonElement("status")]
    [BsonRepresentation(BsonType.String)]
    public UserStatus Status { get; init; } = UserStatus.Active;

    [BsonElement("roles")]
    public List<string> Roles { get; init; } = new() { "user" };

    [BsonElement("preferences")]
    public UserPreferences? Preferences { get; init; }

    [BsonElement("created_at")]
    [BsonDateTimeOptions(Kind = DateTimeKind.Utc)]
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;

    [BsonElement("updated_at")]
    [BsonDateTimeOptions(Kind = DateTimeKind.Utc)]
    public DateTime UpdatedAt { get; init; } = DateTime.UtcNow;

    [BsonElement("last_login")]
    [BsonDateTimeOptions(Kind = DateTimeKind.Utc)]
    public DateTime? LastLogin { get; init; }

    [BsonElement("login_count")]
    public int LoginCount { get; init; } = 0;

    // Méthode helper pour mise à jour
    public User WithIncrementedLogin() => this with
    {
        LoginCount = LoginCount + 1,
        LastLogin = DateTime.UtcNow,
        UpdatedAt = DateTime.UtcNow
    };
}
```

```csharp
// Models/UserStatus.cs
namespace MyApp.Models;

public enum UserStatus
{
    Active,
    Inactive,
    Suspended,
    Deleted
}
```

```csharp
// Models/UserPreferences.cs
using MongoDB.Bson.Serialization.Attributes;

namespace MyApp.Models;

public record UserPreferences
{
    [BsonElement("language")]
    public string Language { get; init; } = "en";

    [BsonElement("timezone")]
    public string Timezone { get; init; } = "UTC";

    [BsonElement("notifications")]
    public bool Notifications { get; init; } = true;
}
```

### DTOs

```csharp
// DTOs/CreateUserDto.cs
using System.ComponentModel.DataAnnotations;

namespace MyApp.DTOs;

public record CreateUserDto
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public required string Name { get; init; }

    [Required]
    [EmailAddress]
    public required string Email { get; init; }

    [Range(18, 120)]
    public int? Age { get; init; }

    [Required]
    [MinLength(8)]
    public required string Password { get; init; }

    public List<string>? Roles { get; init; }
}
```

```csharp
// DTOs/UpdateUserDto.cs
using System.ComponentModel.DataAnnotations;

namespace MyApp.DTOs;

public record UpdateUserDto
{
    [StringLength(100, MinimumLength = 2)]
    public string? Name { get; init; }

    [EmailAddress]
    public string? Email { get; init; }

    [Range(18, 120)]
    public int? Age { get; init; }
}
```

```csharp
// DTOs/UserPublicDto.cs
using MyApp.Models;

namespace MyApp.DTOs;

public record UserPublicDto
{
    public required string Id { get; init; }
    public required string Name { get; init; }
    public required string Email { get; init; }
    public required UserStatus Status { get; init; }
    public required DateTime CreatedAt { get; init; }
}
```

## Repository Pattern

### Interface de base

```csharp
// Repositories/IBaseRepository.cs
using MongoDB.Driver;
using System.Linq.Expressions;

namespace MyApp.Repositories;

public interface IBaseRepository<T> where T : class
{
    Task<T?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
    Task<T?> FindOneAsync(FilterDefinition<T> filter, CancellationToken cancellationToken = default);
    Task<List<T>> FindManyAsync(FilterDefinition<T> filter, CancellationToken cancellationToken = default);
    Task<List<T>> FindManyAsync(FilterDefinition<T> filter, int limit, int skip, CancellationToken cancellationToken = default);
    Task<List<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<T> InsertOneAsync(T document, CancellationToken cancellationToken = default);
    Task<bool> UpdateOneAsync(FilterDefinition<T> filter, UpdateDefinition<T> update, CancellationToken cancellationToken = default);
    Task<bool> UpdateByIdAsync(string id, UpdateDefinition<T> update, CancellationToken cancellationToken = default);
    Task<bool> DeleteOneAsync(FilterDefinition<T> filter, CancellationToken cancellationToken = default);
    Task<bool> DeleteByIdAsync(string id, CancellationToken cancellationToken = default);
    Task<long> CountAsync(FilterDefinition<T> filter, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(FilterDefinition<T> filter, CancellationToken cancellationToken = default);
}
```

### Implémentation de base

```csharp
// Repositories/BaseRepository.cs
using MongoDB.Bson;
using MongoDB.Driver;

namespace MyApp.Repositories;

public abstract class BaseRepository<T> : IBaseRepository<T> where T : class
{
    protected readonly IMongoCollection<T> Collection;
    protected readonly ILogger<BaseRepository<T>> Logger;

    protected BaseRepository(
        IMongoDatabase database,
        string collectionName,
        ILogger<BaseRepository<T>> logger)
    {
        Collection = database.GetCollection<T>(collectionName);
        Logger = logger;
    }

    public virtual async Task<T?> GetByIdAsync(
        string id,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        return await Collection.Find(filter).FirstOrDefaultAsync(cancellationToken);
    }

    public virtual async Task<T?> FindOneAsync(
        FilterDefinition<T> filter,
        CancellationToken cancellationToken = default)
    {
        return await Collection.Find(filter).FirstOrDefaultAsync(cancellationToken);
    }

    public virtual async Task<List<T>> FindManyAsync(
        FilterDefinition<T> filter,
        CancellationToken cancellationToken = default)
    {
        return await Collection.Find(filter).ToListAsync(cancellationToken);
    }

    public virtual async Task<List<T>> FindManyAsync(
        FilterDefinition<T> filter,
        int limit,
        int skip,
        CancellationToken cancellationToken = default)
    {
        return await Collection.Find(filter)
            .Limit(limit)
            .Skip(skip)
            .ToListAsync(cancellationToken);
    }

    public virtual async Task<List<T>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        return await Collection.Find(Builders<T>.Filter.Empty)
            .ToListAsync(cancellationToken);
    }

    public virtual async Task<T> InsertOneAsync(
        T document,
        CancellationToken cancellationToken = default)
    {
        await Collection.InsertOneAsync(document, cancellationToken: cancellationToken);
        return document;
    }

    public virtual async Task<bool> UpdateOneAsync(
        FilterDefinition<T> filter,
        UpdateDefinition<T> update,
        CancellationToken cancellationToken = default)
    {
        var result = await Collection.UpdateOneAsync(filter, update, cancellationToken: cancellationToken);
        return result.ModifiedCount > 0;
    }

    public virtual async Task<bool> UpdateByIdAsync(
        string id,
        UpdateDefinition<T> update,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        return await UpdateOneAsync(filter, update, cancellationToken);
    }

    public virtual async Task<bool> DeleteOneAsync(
        FilterDefinition<T> filter,
        CancellationToken cancellationToken = default)
    {
        var result = await Collection.DeleteOneAsync(filter, cancellationToken);
        return result.DeletedCount > 0;
    }

    public virtual async Task<bool> DeleteByIdAsync(
        string id,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        return await DeleteOneAsync(filter, cancellationToken);
    }

    public virtual async Task<long> CountAsync(
        FilterDefinition<T> filter,
        CancellationToken cancellationToken = default)
    {
        return await Collection.CountDocumentsAsync(filter, cancellationToken: cancellationToken);
    }

    public virtual async Task<bool> ExistsAsync(
        FilterDefinition<T> filter,
        CancellationToken cancellationToken = default)
    {
        var count = await Collection.CountDocumentsAsync(
            filter,
            new CountOptions { Limit = 1 },
            cancellationToken
        );
        return count > 0;
    }
}
```

### Repository utilisateur

```csharp
// Repositories/IUserRepository.cs
using MyApp.Models;

namespace MyApp.Repositories;

public interface IUserRepository : IBaseRepository<User>
{
    Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default);
    Task<List<User>> GetActiveUsersAsync(int limit, int skip, CancellationToken cancellationToken = default);
    Task IncrementLoginCountAsync(string userId, CancellationToken cancellationToken = default);
    Task<List<User>> SearchUsersAsync(string searchTerm, int limit, CancellationToken cancellationToken = default);
    Task<List<User>> GetByRoleAsync(string role, CancellationToken cancellationToken = default);
    Task<int> BulkUpdateStatusAsync(List<string> userIds, UserStatus status, CancellationToken cancellationToken = default);
    Task<UserStatistics> GetStatisticsAsync(CancellationToken cancellationToken = default);
}
```

```csharp
// Repositories/UserRepository.cs
using MongoDB.Driver;
using MongoDB.Bson;
using MyApp.Models;

namespace MyApp.Repositories;

public class UserRepository : BaseRepository<User>, IUserRepository
{
    public UserRepository(
        IMongoDatabase database,
        ILogger<UserRepository> logger)
        : base(database, "users", logger)
    {
        EnsureIndexesAsync().GetAwaiter().GetResult();
    }

    private async Task EnsureIndexesAsync()
    {
        // Index unique sur email
        var emailIndexModel = new CreateIndexModel<User>(
            Builders<User>.IndexKeys.Ascending(u => u.Email),
            new CreateIndexOptions { Unique = true }
        );

        // Index sur status
        var statusIndexModel = new CreateIndexModel<User>(
            Builders<User>.IndexKeys.Ascending(u => u.Status)
        );

        // Index composé
        var compoundIndexModel = new CreateIndexModel<User>(
            Builders<User>.IndexKeys
                .Ascending(u => u.Status)
                .Descending(u => u.CreatedAt)
        );

        await Collection.Indexes.CreateManyAsync(new[]
        {
            emailIndexModel,
            statusIndexModel,
            compoundIndexModel
        });

        Logger.LogInformation("Index créés pour la collection users");
    }

    public async Task<User?> GetByEmailAsync(
        string email,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<User>.Filter.Eq(u => u.Email, email);
        return await FindOneAsync(filter, cancellationToken);
    }

    public async Task<List<User>> GetActiveUsersAsync(
        int limit,
        int skip,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<User>.Filter.Eq(u => u.Status, UserStatus.Active);

        return await Collection.Find(filter)
            .SortByDescending(u => u.CreatedAt)
            .Limit(limit)
            .Skip(skip)
            .ToListAsync(cancellationToken);
    }

    public async Task IncrementLoginCountAsync(
        string userId,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<User>.Filter.Eq("_id", ObjectId.Parse(userId));
        var update = Builders<User>.Update
            .Inc(u => u.LoginCount, 1)
            .Set(u => u.LastLogin, DateTime.UtcNow)
            .Set(u => u.UpdatedAt, DateTime.UtcNow);

        await Collection.UpdateOneAsync(filter, update, cancellationToken: cancellationToken);
    }

    public async Task<List<User>> SearchUsersAsync(
        string searchTerm,
        int limit,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<User>.Filter.Or(
            Builders<User>.Filter.Regex(u => u.Name, new BsonRegularExpression(searchTerm, "i")),
            Builders<User>.Filter.Regex(u => u.Email, new BsonRegularExpression(searchTerm, "i"))
        );

        return await Collection.Find(filter)
            .Limit(limit)
            .ToListAsync(cancellationToken);
    }

    public async Task<List<User>> GetByRoleAsync(
        string role,
        CancellationToken cancellationToken = default)
    {
        var filter = Builders<User>.Filter.AnyEq(u => u.Roles, role);
        return await FindManyAsync(filter, cancellationToken);
    }

    public async Task<int> BulkUpdateStatusAsync(
        List<string> userIds,
        UserStatus status,
        CancellationToken cancellationToken = default)
    {
        var objectIds = userIds.Select(id => ObjectId.Parse(id)).ToList();
        var filter = Builders<User>.Filter.In("_id", objectIds);
        var update = Builders<User>.Update
            .Set(u => u.Status, status)
            .Set(u => u.UpdatedAt, DateTime.UtcNow);

        var result = await Collection.UpdateManyAsync(filter, update, cancellationToken: cancellationToken);
        return (int)result.ModifiedCount;
    }

    public async Task<UserStatistics> GetStatisticsAsync(
        CancellationToken cancellationToken = default)
    {
        // Total
        var total = await CountAsync(Builders<User>.Filter.Empty, cancellationToken);

        // Par status
        var byStatusPipeline = new[]
        {
            new BsonDocument("$group", new BsonDocument
            {
                { "_id", "$status" },
                { "count", new BsonDocument("$sum", 1) }
            })
        };

        var byStatusResult = await Collection
            .Aggregate<BsonDocument>(byStatusPipeline)
            .ToListAsync(cancellationToken);

        var byStatus = byStatusResult.ToDictionary(
            doc => doc["_id"].AsString,
            doc => doc["count"].AsInt32
        );

        // Inscriptions récentes
        var thirtyDaysAgo = DateTime.UtcNow.AddDays(-30);
        var recentFilter = Builders<User>.Filter.Gte(u => u.CreatedAt, thirtyDaysAgo);
        var recentSignups = await CountAsync(recentFilter, cancellationToken);

        return new UserStatistics
        {
            Total = total,
            ByStatus = byStatus,
            RecentSignups = recentSignups
        };
    }
}

public class UserStatistics
{
    public long Total { get; init; }
    public Dictionary<string, int> ByStatus { get; init; } = new();
    public long RecentSignups { get; init; }
}
```

## Service Layer

```csharp
// Services/IUserService.cs
using MyApp.DTOs;
using MyApp.Models;

namespace MyApp.Services;

public interface IUserService
{
    Task<UserPublicDto> RegisterUserAsync(CreateUserDto dto, CancellationToken cancellationToken = default);
    Task<User> AuthenticateUserAsync(string email, string password, CancellationToken cancellationToken = default);
    Task<UserPublicDto> GetUserByIdAsync(string userId, CancellationToken cancellationToken = default);
    Task<UserPublicDto> UpdateUserAsync(string userId, UpdateUserDto dto, CancellationToken cancellationToken = default);
    Task<List<UserPublicDto>> SearchUsersAsync(string query, int limit, CancellationToken cancellationToken = default);
    Task<UserStatistics> GetDashboardStatsAsync(CancellationToken cancellationToken = default);
    Task<bool> SuspendUserAsync(string userId, CancellationToken cancellationToken = default);
}
```

```csharp
// Services/UserService.cs
using MyApp.DTOs;
using MyApp.Models;
using MyApp.Repositories;
using MongoDB.Driver;

namespace MyApp.Services;

public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;

    public UserService(
        IUserRepository repository,
        ILogger<UserService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<UserPublicDto> RegisterUserAsync(
        CreateUserDto dto,
        CancellationToken cancellationToken = default)
    {
        // Vérifier si l'email existe
        var existing = await _repository.GetByEmailAsync(dto.Email, cancellationToken);
        if (existing is not null)
        {
            throw new InvalidOperationException($"Email {dto.Email} already registered");
        }

        // Créer l'utilisateur
        var user = new User
        {
            Name = dto.Name,
            Email = dto.Email,
            Age = dto.Age,
            Status = UserStatus.Active,
            Roles = dto.Roles ?? new List<string> { "user" },
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow,
            LoginCount = 0
        };

        try
        {
            user = await _repository.InsertOneAsync(user, cancellationToken);
            _logger.LogInformation("New user registered: {Email}", user.Email);

            return ConvertToPublicDto(user);
        }
        catch (MongoWriteException ex) when (ex.WriteError.Category == ServerErrorCategory.DuplicateKey)
        {
            throw new InvalidOperationException("Email already exists");
        }
    }

    public async Task<User> AuthenticateUserAsync(
        string email,
        string password,
        CancellationToken cancellationToken = default)
    {
        var user = await _repository.GetByEmailAsync(email, cancellationToken)
            ?? throw new InvalidOperationException("Invalid credentials");

        if (user.Status != UserStatus.Active)
        {
            throw new InvalidOperationException("Account is not active");
        }

        // TODO: Vérifier le mot de passe hashé

        // Incrémenter le compteur
        await _repository.IncrementLoginCountAsync(user.Id!, cancellationToken);

        _logger.LogInformation("User logged in: {Email}", user.Email);
        return user;
    }

    public async Task<UserPublicDto> GetUserByIdAsync(
        string userId,
        CancellationToken cancellationToken = default)
    {
        var user = await _repository.GetByIdAsync(userId, cancellationToken)
            ?? throw new KeyNotFoundException("User not found");

        return ConvertToPublicDto(user);
    }

    public async Task<UserPublicDto> UpdateUserAsync(
        string userId,
        UpdateUserDto dto,
        CancellationToken cancellationToken = default)
    {
        var updateBuilder = Builders<User>.Update.Set(u => u.UpdatedAt, DateTime.UtcNow);

        if (dto.Name is not null)
            updateBuilder = updateBuilder.Set(u => u.Name, dto.Name);

        if (dto.Email is not null)
            updateBuilder = updateBuilder.Set(u => u.Email, dto.Email);

        if (dto.Age is not null)
            updateBuilder = updateBuilder.Set(u => u.Age, dto.Age);

        var success = await _repository.UpdateByIdAsync(userId, updateBuilder, cancellationToken);

        if (!success)
        {
            throw new InvalidOperationException("Failed to update user");
        }

        return await GetUserByIdAsync(userId, cancellationToken);
    }

    public async Task<List<UserPublicDto>> SearchUsersAsync(
        string query,
        int limit,
        CancellationToken cancellationToken = default)
    {
        var users = await _repository.SearchUsersAsync(query, limit, cancellationToken);
        return users.Select(ConvertToPublicDto).ToList();
    }

    public async Task<UserStatistics> GetDashboardStatsAsync(
        CancellationToken cancellationToken = default)
    {
        return await _repository.GetStatisticsAsync(cancellationToken);
    }

    public async Task<bool> SuspendUserAsync(
        string userId,
        CancellationToken cancellationToken = default)
    {
        var update = Builders<User>.Update
            .Set(u => u.Status, UserStatus.Suspended)
            .Set(u => u.UpdatedAt, DateTime.UtcNow);

        return await _repository.UpdateByIdAsync(userId, update, cancellationToken);
    }

    private static UserPublicDto ConvertToPublicDto(User user) => new()
    {
        Id = user.Id!,
        Name = user.Name,
        Email = user.Email,
        Status = user.Status,
        CreatedAt = user.CreatedAt
    };
}
```

## Opérations avancées avec LINQ

### Recherche LINQ

```csharp
// Exemple de recherche avec LINQ to MongoDB
public async Task<List<User>> GetUsersByAgeRangeAsync(int minAge, int maxAge)
{
    return await Collection.AsQueryable()
        .Where(u => u.Age >= minAge && u.Age <= maxAge && u.Status == UserStatus.Active)
        .OrderByDescending(u => u.CreatedAt)
        .Take(100)
        .ToListAsync();
}

// Grouping avec LINQ
public async Task<List<AgeGroup>> GetUsersByAgeGroupAsync()
{
    return await Collection.AsQueryable()
        .Where(u => u.Age.HasValue)
        .GroupBy(u => u.Age.Value / 10 * 10) // Grouper par décennie
        .Select(g => new AgeGroup
        {
            AgeRange = g.Key,
            Count = g.Count(),
            AverageAge = g.Average(u => u.Age!.Value)
        })
        .OrderBy(g => g.AgeRange)
        .ToListAsync();
}

public class AgeGroup
{
    public int AgeRange { get; set; }
    public int Count { get; set; }
    public double AverageAge { get; set; }
}
```

### Pagination avec LINQ

```csharp
public class PaginatedResult<T>
{
    public List<T> Data { get; init; } = new();
    public long Total { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalPages { get; init; }
}

public async Task<PaginatedResult<User>> GetUsersPaginatedAsync(
    int page,
    int pageSize,
    CancellationToken cancellationToken = default)
{
    var query = Collection.AsQueryable()
        .Where(u => u.Status == UserStatus.Active);

    var total = await query.CountAsync(cancellationToken);

    var data = await query
        .OrderByDescending(u => u.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync(cancellationToken);

    return new PaginatedResult<User>
    {
        Data = data,
        Total = total,
        Page = page,
        PageSize = pageSize,
        TotalPages = (int)Math.Ceiling((double)total / pageSize)
    };
}
```

## Agrégations

```csharp
// service/AnalyticsService.cs
public class AnalyticsService
{
    private readonly IMongoCollection<BsonDocument> _ordersCollection;

    public async Task<SalesAnalytics> GetSalesAnalyticsAsync(
        DateTime startDate,
        DateTime endDate,
        CancellationToken cancellationToken = default)
    {
        var pipeline = new[]
        {
            // Filtrer par période
            new BsonDocument("$match", new BsonDocument
            {
                { "createdAt", new BsonDocument
                    {
                        { "$gte", startDate },
                        { "$lte", endDate }
                    }
                },
                { "status", "completed" }
            }),

            // Dérouler les items
            new BsonDocument("$unwind", "$items"),

            // Jointure avec products
            new BsonDocument("$lookup", new BsonDocument
            {
                { "from", "products" },
                { "localField", "items.productId" },
                { "foreignField", "_id" },
                { "as", "productDetails" }
            }),

            new BsonDocument("$unwind", "$productDetails"),

            // Grouper par catégorie
            new BsonDocument("$group", new BsonDocument
            {
                { "_id", "$productDetails.category" },
                { "totalSales", new BsonDocument("$sum", new BsonDocument("$multiply", new BsonArray
                    {
                        "$items.quantity",
                        "$items.price"
                    }))
                },
                { "totalItems", new BsonDocument("$sum", "$items.quantity") },
                { "avgOrderValue", new BsonDocument("$avg", "$totalAmount") }
            }),

            // Trier
            new BsonDocument("$sort", new BsonDocument("totalSales", -1))
        };

        var results = await _ordersCollection
            .Aggregate<BsonDocument>(pipeline)
            .ToListAsync(cancellationToken);

        return new SalesAnalytics
        {
            StartDate = startDate,
            EndDate = endDate,
            ByCategory = results.Select(doc => new CategorySales
            {
                Category = doc["_id"].AsString,
                TotalSales = doc["totalSales"].AsDouble,
                TotalItems = doc["totalItems"].AsInt32,
                AverageOrderValue = doc["avgOrderValue"].AsDouble
            }).ToList()
        };
    }

    public async Task<ProductDashboard> GetProductDashboardAsync(
        CancellationToken cancellationToken = default)
    {
        var pipeline = new[]
        {
            new BsonDocument("$facet", new BsonDocument
            {
                { "generalStats", new BsonArray
                    {
                        new BsonDocument("$group", new BsonDocument
                        {
                            { "_id", BsonNull.Value },
                            { "totalProducts", new BsonDocument("$sum", 1) },
                            { "avgPrice", new BsonDocument("$avg", "$price.amount") },
                            { "totalInventory", new BsonDocument("$sum", "$inventory.quantity") }
                        })
                    }
                },
                { "byCategory", new BsonArray
                    {
                        new BsonDocument("$group", new BsonDocument
                        {
                            { "_id", "$category" },
                            { "count", new BsonDocument("$sum", 1) },
                            { "avgPrice", new BsonDocument("$avg", "$price.amount") }
                        }),
                        new BsonDocument("$sort", new BsonDocument("count", -1))
                    }
                },
                { "topProducts", new BsonArray
                    {
                        new BsonDocument("$sort", new BsonDocument("reviews.rating", -1)),
                        new BsonDocument("$limit", 10),
                        new BsonDocument("$project", new BsonDocument
                        {
                            { "name", 1 },
                            { "price", "$price.amount" },
                            { "rating", "$reviews.rating" }
                        })
                    }
                }
            })
        };

        var result = await _productsCollection
            .Aggregate<BsonDocument>(pipeline)
            .FirstOrDefaultAsync(cancellationToken);

        return new ProductDashboard(result);
    }
}

public class SalesAnalytics
{
    public DateTime StartDate { get; init; }
    public DateTime EndDate { get; init; }
    public List<CategorySales> ByCategory { get; init; } = new();
}

public class CategorySales
{
    public required string Category { get; init; }
    public double TotalSales { get; init; }
    public int TotalItems { get; init; }
    public double AverageOrderValue { get; init; }
}
```

## Transactions

```csharp
// Services/TransactionService.cs
public class TransactionService
{
    private readonly IMongoClient _client;
    private readonly IMongoDatabase _database;
    private readonly ILogger<TransactionService> _logger;

    public TransactionService(
        IMongoClient client,
        IMongoDatabase database,
        ILogger<TransactionService> logger)
    {
        _client = client;
        _database = database;
        _logger = logger;
    }

    public async Task TransferFundsAsync(
        string fromAccountId,
        string toAccountId,
        decimal amount,
        CancellationToken cancellationToken = default)
    {
        using var session = await _client.StartSessionAsync(cancellationToken: cancellationToken);

        session.StartTransaction(new TransactionOptions(
            readPreference: ReadPreference.Primary,
            readConcern: ReadConcern.Snapshot,
            writeConcern: WriteConcern.WMajority
        ));

        try
        {
            var accounts = _database.GetCollection<BsonDocument>("accounts");

            // Débiter
            var debitFilter = Builders<BsonDocument>.Filter.And(
                Builders<BsonDocument>.Filter.Eq("_id", ObjectId.Parse(fromAccountId)),
                Builders<BsonDocument>.Filter.Gte("balance", amount)
            );

            var debitUpdate = Builders<BsonDocument>.Update
                .Inc("balance", -amount)
                .Push("transactions", new BsonDocument
                {
                    { "type", "debit" },
                    { "amount", amount },
                    { "timestamp", DateTime.UtcNow }
                });

            var debitResult = await accounts.UpdateOneAsync(
                session,
                debitFilter,
                debitUpdate,
                cancellationToken: cancellationToken
            );

            if (debitResult.ModifiedCount == 0)
            {
                throw new InvalidOperationException("Insufficient funds or account not found");
            }

            // Créditer
            var creditFilter = Builders<BsonDocument>.Filter.Eq("_id", ObjectId.Parse(toAccountId));
            var creditUpdate = Builders<BsonDocument>.Update
                .Inc("balance", amount)
                .Push("transactions", new BsonDocument
                {
                    { "type", "credit" },
                    { "amount", amount },
                    { "timestamp", DateTime.UtcNow }
                });

            await accounts.UpdateOneAsync(
                session,
                creditFilter,
                creditUpdate,
                cancellationToken: cancellationToken
            );

            // Commit
            await session.CommitTransactionAsync(cancellationToken);
            _logger.LogInformation("✅ Transaction completed successfully");
        }
        catch (Exception ex)
        {
            await session.AbortTransactionAsync(cancellationToken);
            _logger.LogError(ex, "❌ Transaction failed");
            throw;
        }
    }

    public async Task<string> CreateOrderWithInventoryUpdateAsync(
        OrderData orderData,
        CancellationToken cancellationToken = default)
    {
        const int maxRetries = 3;
        Exception? lastException = null;

        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            using var session = await _client.StartSessionAsync(cancellationToken: cancellationToken);

            session.StartTransaction(new TransactionOptions(
                readPreference: ReadPreference.Primary,
                readConcern: ReadConcern.Snapshot,
                writeConcern: WriteConcern.WMajority
            ));

            try
            {
                var orders = _database.GetCollection<BsonDocument>("orders");
                var products = _database.GetCollection<BsonDocument>("products");
                var users = _database.GetCollection<BsonDocument>("users");

                // 1. Réserver l'inventaire
                foreach (var item in orderData.Items)
                {
                    var filter = Builders<BsonDocument>.Filter.And(
                        Builders<BsonDocument>.Filter.Eq("_id", ObjectId.Parse(item.ProductId)),
                        Builders<BsonDocument>.Filter.Gte("inventory.quantity", item.Quantity)
                    );

                    var update = Builders<BsonDocument>.Update
                        .Inc("inventory.quantity", -item.Quantity)
                        .Set("updatedAt", DateTime.UtcNow);

                    var result = await products.UpdateOneAsync(
                        session,
                        filter,
                        update,
                        cancellationToken: cancellationToken
                    );

                    if (result.ModifiedCount == 0)
                    {
                        throw new InvalidOperationException($"Product {item.ProductId} out of stock");
                    }
                }

                // 2. Créer la commande
                var order = new BsonDocument
                {
                    { "userId", ObjectId.Parse(orderData.UserId) },
                    { "items", new BsonArray(orderData.Items.Select(i => i.ToBsonDocument())) },
                    { "status", "pending" },
                    { "totalAmount", orderData.TotalAmount },
                    { "createdAt", DateTime.UtcNow },
                    { "updatedAt", DateTime.UtcNow }
                };

                await orders.InsertOneAsync(session, order, cancellationToken: cancellationToken);
                var orderId = order["_id"].AsObjectId.ToString();

                // 3. Mettre à jour les stats utilisateur
                await users.UpdateOneAsync(
                    session,
                    Builders<BsonDocument>.Filter.Eq("_id", ObjectId.Parse(orderData.UserId)),
                    Builders<BsonDocument>.Update
                        .Inc("stats.totalOrders", 1)
                        .Set("stats.lastOrderDate", DateTime.UtcNow),
                    cancellationToken: cancellationToken
                );

                // Commit
                await session.CommitTransactionAsync(cancellationToken);
                _logger.LogInformation("✅ Order created: {OrderId}", orderId);

                return orderId;
            }
            catch (MongoException ex) when (ex.HasErrorLabel("TransientTransactionError") && attempt < maxRetries)
            {
                await session.AbortTransactionAsync(cancellationToken);
                lastException = ex;
                _logger.LogWarning("Transaction failed (attempt {Attempt}), retrying...", attempt);
                await Task.Delay(100 * attempt, cancellationToken);
                continue;
            }
            catch (Exception ex)
            {
                await session.AbortTransactionAsync(cancellationToken);
                throw;
            }
        }

        throw new InvalidOperationException(
            $"Transaction failed after {maxRetries} attempts",
            lastException
        );
    }
}

public class OrderData
{
    public required string UserId { get; init; }
    public required List<OrderItem> Items { get; init; }
    public decimal TotalAmount { get; init; }
}

public class OrderItem
{
    public required string ProductId { get; init; }
    public int Quantity { get; init; }
    public decimal Price { get; init; }

    public BsonDocument ToBsonDocument() => new()
    {
        { "productId", ObjectId.Parse(ProductId) },
        { "quantity", Quantity },
        { "price", Price }
    };
}
```

## Controller ASP.NET Core

```csharp
// Controllers/UsersController.cs
using Microsoft.AspNetCore.Mvc;
using MyApp.DTOs;
using MyApp.Services;

namespace MyApp.Controllers;

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    public UsersController(
        IUserService userService,
        ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [HttpPost]
    [ProducesResponseType(typeof(UserPublicDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateUser(
        [FromBody] CreateUserDto dto,
        CancellationToken cancellationToken)
    {
        try
        {
            var user = await _userService.RegisterUserAsync(dto, cancellationToken);
            return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
        }
        catch (InvalidOperationException ex)
        {
            return BadRequest(new { error = ex.Message });
        }
    }

    [HttpGet("{id}")]
    [ProducesResponseType(typeof(UserPublicDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetUser(
        string id,
        CancellationToken cancellationToken)
    {
        try
        {
            var user = await _userService.GetUserByIdAsync(id, cancellationToken);
            return Ok(user);
        }
        catch (KeyNotFoundException)
        {
            return NotFound();
        }
    }

    [HttpPut("{id}")]
    [ProducesResponseType(typeof(UserPublicDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateUser(
        string id,
        [FromBody] UpdateUserDto dto,
        CancellationToken cancellationToken)
    {
        try
        {
            var user = await _userService.UpdateUserAsync(id, dto, cancellationToken);
            return Ok(user);
        }
        catch (KeyNotFoundException)
        {
            return NotFound();
        }
        catch (InvalidOperationException ex)
        {
            return BadRequest(new { error = ex.Message });
        }
    }

    [HttpGet("search")]
    [ProducesResponseType(typeof(List<UserPublicDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> SearchUsers(
        [FromQuery] string query,
        [FromQuery] int limit = 20,
        CancellationToken cancellationToken = default)
    {
        var users = await _userService.SearchUsersAsync(query, limit, cancellationToken);
        return Ok(users);
    }

    [HttpGet("stats")]
    [ProducesResponseType(typeof(UserStatistics), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetStatistics(
        CancellationToken cancellationToken)
    {
        var stats = await _userService.GetDashboardStatsAsync(cancellationToken);
        return Ok(stats);
    }
}
```

## Gestion d'erreurs

```csharp
// Middleware/ErrorHandlingMiddleware.cs
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;

    public ErrorHandlingMiddleware(
        RequestDelegate next,
        ILogger<ErrorHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (MongoWriteException ex) when (ex.WriteError.Category == ServerErrorCategory.DuplicateKey)
        {
            _logger.LogWarning("Duplicate key error: {Message}", ex.Message);

            context.Response.StatusCode = StatusCodes.Status409Conflict;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Duplicate key error",
                message = "A record with this key already exists"
            });
        }
        catch (MongoException ex)
        {
            _logger.LogError(ex, "MongoDB error occurred");

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Database error",
                message = "An error occurred while processing your request"
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Internal server error",
                message = "An unexpected error occurred"
            });
        }
    }
}
```

## Bonnes pratiques

### ✅ DO - À faire

```csharp
// 1. Utiliser records pour les DTOs immutables
public record UserDto(string Name, string Email);

// 2. Utiliser nullable reference types
#nullable enable
public string? OptionalField { get; init; }

// 3. Utiliser CancellationToken
public async Task<User> GetUserAsync(CancellationToken cancellationToken = default)

// 4. Typer les collections
IMongoCollection<User> users = database.GetCollection<User>("users");

// 5. Utiliser dependency injection
services.AddScoped<IUserService, UserService>();

// 6. Gérer les erreurs spécifiques
catch (MongoWriteException ex) when (ex.WriteError.Category == ServerErrorCategory.DuplicateKey)

// 7. Utiliser LINQ quand approprié
var users = await collection.AsQueryable()
    .Where(u => u.Status == UserStatus.Active)
    .ToListAsync();

// 8. Utiliser async/await partout
public async Task<User> GetUserAsync(string id)

// 9. Logger les opérations importantes
_logger.LogInformation("User created: {UserId}", user.Id);

// 10. Utiliser des attributs BSON
[BsonElement("created_at")]
[BsonDateTimeOptions(Kind = DateTimeKind.Utc)]
public DateTime CreatedAt { get; init; }
```

### ❌ DON'T - À éviter

```csharp
// ❌ Ne pas créer un nouveau client par requête
// ❌ Ne pas bloquer sur des opérations async (.Result, .Wait())
// ❌ Ne pas ignorer CancellationToken
// ❌ Ne pas utiliser Document partout
// ❌ Ne pas ignorer les erreurs
// ❌ Ne pas exposer les entités directement dans les APIs
// ❌ Ne pas faire de requêtes N+1
// ❌ Ne pas oublier les index
```

## Comparaison des approches

| Approche | Avantages | Inconvénients |
|----------|-----------|---------------|
| **Builders API** | Type-safe, Performant | Verbeux |
| **LINQ** | Lisible, Familier C# | Peut générer des requêtes non optimales |
| **BsonDocument** | Flexible | Pas de type safety |
| **Records** | Immutable, Concis | Moins flexible pour mutations |

## Conclusion

Le driver C# MongoDB offre une expérience .NET de première classe :

1. **Type Safety** : Typage fort avec génériques et nullable
2. **Async/Await** : Support async natif et performant
3. **LINQ** : Requêtes familières pour développeurs C#
4. **DI** : Intégration parfaite avec ASP.NET Core
5. **Records** : Modèles immutables modernes
6. **Performance** : Parmi les meilleurs drivers

**Points clés** :
- Singleton pour IMongoClient
- Dependency Injection
- CancellationToken partout
- Records pour immutabilité
- LINQ pour requêtes simples
- Builders pour performance
- Gestion d'erreurs robuste

---


⏭️ [Driver Go](/15-drivers-integration-applicative/06-driver-go.md)
