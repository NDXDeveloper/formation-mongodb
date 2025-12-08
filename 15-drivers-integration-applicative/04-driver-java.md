🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.4 Driver Java

## Introduction

Le driver Java MongoDB est l'un des drivers les plus matures et performants de l'écosystème MongoDB. Il offre une API robuste, typée, et s'intègre parfaitement avec l'écosystème Java Enterprise. Cette section couvre le **driver synchrone**, le **driver reactive streams**, et **Spring Data MongoDB**.

## Installation et configuration

### Maven

```xml
<!-- pom.xml -->
<project>
    <properties>
        <mongodb.driver.version>5.1.0</mongodb.driver.version>
        <spring.data.mongodb.version>4.2.0</spring.data.mongodb.version>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Driver synchrone -->
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongodb-driver-sync</artifactId>
            <version>${mongodb.driver.version}</version>
        </dependency>

        <!-- Driver reactive streams -->
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongodb-driver-reactivestreams</artifactId>
            <version>${mongodb.driver.version}</version>
        </dependency>

        <!-- BSON -->
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>bson</artifactId>
            <version>${mongodb.driver.version}</version>
        </dependency>

        <!-- Spring Data MongoDB (optionnel) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
            <version>3.2.0</version>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>jakarta.validation</groupId>
            <artifactId>jakarta.validation-api</artifactId>
            <version>3.0.2</version>
        </dependency>

        <!-- Lombok (optionnel mais recommandé) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.30</version>
            <scope>provided</scope>
        </dependency>

        <!-- SLF4J pour logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.9</version>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.4.14</version>
        </dependency>
    </dependencies>
</project>
```

### Gradle

```groovy
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // Driver synchrone
    implementation 'org.mongodb:mongodb-driver-sync:5.1.0'

    // Driver reactive
    implementation 'org.mongodb:mongodb-driver-reactivestreams:5.1.0'

    // Spring Data MongoDB
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'

    // Validation
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // Test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

## Configuration de connexion

### Configuration centralisée

```java
// config/MongoDBConfig.java
package com.example.config;

import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.ServerApi;
import com.mongodb.ServerApiVersion;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import org.bson.codecs.configuration.CodecRegistry;
import org.bson.codecs.pojo.PojoCodecProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.TimeUnit;

import static org.bson.codecs.configuration.CodecRegistries.fromProviders;
import static org.bson.codecs.configuration.CodecRegistries.fromRegistries;
import static com.mongodb.MongoClientSettings.getDefaultCodecRegistry;

public class MongoDBConfig {

    private static final Logger logger = LoggerFactory.getLogger(MongoDBConfig.class);

    private final String connectionString;
    private final String databaseName;

    public MongoDBConfig() {
        this.connectionString = System.getenv()
            .getOrDefault("MONGODB_URI", "mongodb://localhost:27017");
        this.databaseName = System.getenv()
            .getOrDefault("MONGODB_DATABASE", "myapp");
    }

    public MongoClientSettings getClientSettings() {
        // Codec registry pour POJO mapping
        CodecRegistry pojoCodecRegistry = fromRegistries(
            getDefaultCodecRegistry(),
            fromProviders(PojoCodecProvider.builder()
                .automatic(true)
                .build())
        );

        // Configuration du client
        return MongoClientSettings.builder()
            .applyConnectionString(new ConnectionString(connectionString))
            .codecRegistry(pojoCodecRegistry)

            // Server API (Stable API)
            .serverApi(ServerApi.builder()
                .version(ServerApiVersion.V1)
                .build())

            // Connection Pool
            .applyToConnectionPoolSettings(builder -> builder
                .maxSize(100)
                .minSize(10)
                .maxWaitTime(30, TimeUnit.SECONDS)
                .maxConnectionIdleTime(60, TimeUnit.SECONDS)
                .maxConnectionLifeTime(30, TimeUnit.MINUTES)
            )

            // Socket Settings
            .applyToSocketSettings(builder -> builder
                .connectTimeout(10, TimeUnit.SECONDS)
                .readTimeout(45, TimeUnit.SECONDS)
            )

            // Server Settings
            .applyToServerSettings(builder -> builder
                .heartbeatFrequency(10, TimeUnit.SECONDS)
                .minHeartbeatFrequency(500, TimeUnit.MILLISECONDS)
            )

            // Retry
            .retryWrites(true)
            .retryReads(true)

            .build();
    }

    public String getDatabaseName() {
        return databaseName;
    }
}
```

### Connection Manager (Singleton)

```java
// database/MongoDBConnection.java
package com.example.database;

import com.example.config.MongoDBConfig;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MongoDBConnection {

    private static final Logger logger = LoggerFactory.getLogger(MongoDBConnection.class);
    private static MongoDBConnection instance;

    private final MongoClient client;
    private final MongoDatabase database;
    private final MongoDBConfig config;

    private MongoDBConnection() {
        this.config = new MongoDBConfig();
        this.client = MongoClients.create(config.getClientSettings());
        this.database = client.getDatabase(config.getDatabaseName());

        // Vérifier la connexion
        try {
            database.runCommand(new Document("ping", 1));
            logger.info("✅ Connecté à MongoDB");
        } catch (Exception e) {
            logger.error("❌ Échec de connexion MongoDB", e);
            throw new RuntimeException("Failed to connect to MongoDB", e);
        }
    }

    public static synchronized MongoDBConnection getInstance() {
        if (instance == null) {
            instance = new MongoDBConnection();
        }
        return instance;
    }

    public MongoClient getClient() {
        return client;
    }

    public MongoDatabase getDatabase() {
        return database;
    }

    public void close() {
        if (client != null) {
            client.close();
            logger.info("🔌 Connexion MongoDB fermée");
        }
    }

    // Shutdown hook
    static {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            if (instance != null) {
                instance.close();
            }
        }));
    }
}
```

## Modèles de données (POJOs)

### Modèle utilisateur

```java
// models/User.java
package com.example.models;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.bson.types.ObjectId;

import jakarta.validation.constraints.*;
import java.time.Instant;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    private ObjectId id;

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120, message = "Age must be at most 120")
    private Integer age;

    @NotNull
    @Builder.Default
    private UserStatus status = UserStatus.ACTIVE;

    @NotNull
    @Builder.Default
    private List<String> roles = List.of("user");

    private UserPreferences preferences;

    @Builder.Default
    private Instant createdAt = Instant.now();

    @Builder.Default
    private Instant updatedAt = Instant.now();

    private Instant lastLogin;

    @Builder.Default
    private Integer loginCount = 0;

    // Méthode helper
    public void incrementLoginCount() {
        this.loginCount++;
        this.lastLogin = Instant.now();
        this.updatedAt = Instant.now();
    }
}
```

```java
// models/UserStatus.java
package com.example.models;

public enum UserStatus {
    ACTIVE,
    INACTIVE,
    SUSPENDED,
    DELETED
}
```

```java
// models/UserPreferences.java
package com.example.models;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserPreferences {

    @Builder.Default
    private String language = "en";

    @Builder.Default
    private String timezone = "UTC";

    @Builder.Default
    private Boolean notifications = true;
}
```

### DTOs (Data Transfer Objects)

```java
// dto/CreateUserDTO.java
package com.example.dto;

import jakarta.validation.constraints.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateUserDTO {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @Min(18)
    @Max(120)
    private Integer age;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    private List<String> roles;
}
```

```java
// dto/UpdateUserDTO.java
package com.example.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import jakarta.validation.constraints.*;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UpdateUserDTO {

    @Size(min = 2, max = 100)
    private String name;

    @Email
    private String email;

    @Min(18)
    @Max(120)
    private Integer age;
}
```

```java
// dto/UserPublicDTO.java
package com.example.dto;

import com.example.models.UserStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserPublicDTO {
    private String id;
    private String name;
    private String email;
    private UserStatus status;
    private Instant createdAt;
}
```

## Repository Pattern

### Repository de base

```java
// repository/BaseRepository.java
package com.example.repository;

import com.mongodb.client.MongoCollection;
import com.mongodb.client.model.Filters;
import com.mongodb.client.result.DeleteResult;
import com.mongodb.client.result.InsertOneResult;
import com.mongodb.client.result.UpdateResult;
import org.bson.Document;
import org.bson.conversions.Bson;
import org.bson.types.ObjectId;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public abstract class BaseRepository<T> {

    protected final MongoCollection<T> collection;

    protected BaseRepository(MongoCollection<T> collection) {
        this.collection = collection;
    }

    public Optional<T> findById(ObjectId id) {
        return Optional.ofNullable(
            collection.find(Filters.eq("_id", id)).first()
        );
    }

    public Optional<T> findById(String id) {
        return findById(new ObjectId(id));
    }

    public Optional<T> findOne(Bson filter) {
        return Optional.ofNullable(collection.find(filter).first());
    }

    public List<T> findMany(Bson filter) {
        return collection.find(filter).into(new ArrayList<>());
    }

    public List<T> findMany(Bson filter, int limit, int skip) {
        return collection.find(filter)
            .limit(limit)
            .skip(skip)
            .into(new ArrayList<>());
    }

    public List<T> findAll() {
        return collection.find().into(new ArrayList<>());
    }

    public InsertOneResult insertOne(T document) {
        return collection.insertOne(document);
    }

    public UpdateResult updateOne(Bson filter, Bson update) {
        return collection.updateOne(filter, update);
    }

    public UpdateResult updateById(ObjectId id, Bson update) {
        return updateOne(Filters.eq("_id", id), update);
    }

    public UpdateResult updateById(String id, Bson update) {
        return updateById(new ObjectId(id), update);
    }

    public DeleteResult deleteOne(Bson filter) {
        return collection.deleteOne(filter);
    }

    public DeleteResult deleteById(ObjectId id) {
        return deleteOne(Filters.eq("_id", id));
    }

    public DeleteResult deleteById(String id) {
        return deleteById(new ObjectId(id));
    }

    public long count(Bson filter) {
        return collection.countDocuments(filter);
    }

    public long count() {
        return collection.countDocuments();
    }

    public boolean exists(Bson filter) {
        return collection.countDocuments(filter,
            new com.mongodb.client.model.CountOptions().limit(1)) > 0;
    }
}
```

### Repository utilisateur

```java
// repository/UserRepository.java
package com.example.repository;

import com.example.database.MongoDBConnection;
import com.example.models.User;
import com.example.models.UserStatus;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.model.*;
import com.mongodb.client.result.UpdateResult;
import org.bson.Document;
import org.bson.conversions.Bson;
import org.bson.types.ObjectId;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

public class UserRepository extends BaseRepository<User> {

    private static final Logger logger = LoggerFactory.getLogger(UserRepository.class);

    public UserRepository() {
        super(MongoDBConnection.getInstance()
            .getDatabase()
            .getCollection("users", User.class));
        ensureIndexes();
    }

    private void ensureIndexes() {
        // Index unique sur email
        collection.createIndex(
            Indexes.ascending("email"),
            new IndexOptions().unique(true)
        );

        // Index sur status
        collection.createIndex(Indexes.ascending("status"));

        // Index composé pour recherche et tri
        collection.createIndex(
            Indexes.compoundIndex(
                Indexes.ascending("status"),
                Indexes.descending("createdAt")
            )
        );

        // Index texte pour recherche full-text
        collection.createIndex(
            Indexes.compoundIndex(
                Indexes.text("name"),
                Indexes.text("email")
            )
        );

        logger.info("Index créés pour la collection users");
    }

    public Optional<User> findByEmail(String email) {
        return findOne(Filters.eq("email", email));
    }

    public List<User> findActiveUsers(int limit, int skip) {
        return collection.find(Filters.eq("status", UserStatus.ACTIVE))
            .sort(Sorts.descending("createdAt"))
            .limit(limit)
            .skip(skip)
            .into(new ArrayList<>());
    }

    public boolean updateProfile(ObjectId userId, Document updates) {
        if (updates.isEmpty()) {
            return false;
        }

        updates.append("updatedAt", Instant.now());

        UpdateResult result = updateById(
            userId,
            Updates.combine(
                updates.entrySet().stream()
                    .map(entry -> Updates.set(entry.getKey(), entry.getValue()))
                    .toArray(Bson[]::new)
            )
        );

        return result.getModifiedCount() > 0;
    }

    public void incrementLoginCount(ObjectId userId) {
        updateById(
            userId,
            Updates.combine(
                Updates.inc("loginCount", 1),
                Updates.set("lastLogin", Instant.now()),
                Updates.set("updatedAt", Instant.now())
            )
        );
    }

    public List<User> searchUsers(String searchTerm, int limit) {
        return collection.find(Filters.text(searchTerm))
            .limit(limit)
            .into(new ArrayList<>());
    }

    public List<User> findByRole(String role) {
        return findMany(Filters.in("roles", role));
    }

    public int bulkUpdateStatus(List<ObjectId> userIds, UserStatus status) {
        UpdateResult result = collection.updateMany(
            Filters.in("_id", userIds),
            Updates.combine(
                Updates.set("status", status),
                Updates.set("updatedAt", Instant.now())
            )
        );

        return (int) result.getModifiedCount();
    }

    public UserStatistics getStatistics() {
        // Total
        long total = count();

        // Par status
        List<Document> byStatus = collection.aggregate(Arrays.asList(
            Aggregates.group("$status", Accumulators.sum("count", 1))
        )).into(new ArrayList<>());

        // Inscriptions récentes (30 derniers jours)
        Instant thirtyDaysAgo = Instant.now().minusSeconds(30 * 24 * 60 * 60);
        long recentSignups = count(Filters.gte("createdAt", thirtyDaysAgo));

        return new UserStatistics(total, byStatus, recentSignups);
    }

    // Classe interne pour les statistiques
    public static class UserStatistics {
        public final long total;
        public final List<Document> byStatus;
        public final long recentSignups;

        public UserStatistics(long total, List<Document> byStatus, long recentSignups) {
            this.total = total;
            this.byStatus = byStatus;
            this.recentSignups = recentSignups;
        }
    }
}
```

## Service Layer

```java
// service/UserService.java
package com.example.service;

import com.example.dto.CreateUserDTO;
import com.example.dto.UpdateUserDTO;
import com.example.dto.UserPublicDTO;
import com.example.models.User;
import com.example.models.UserStatus;
import com.example.repository.UserRepository;
import com.mongodb.MongoWriteException;
import org.bson.Document;
import org.bson.types.ObjectId;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.List;
import java.util.stream.Collectors;

public class UserService {

    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public UserPublicDTO registerUser(CreateUserDTO dto) {
        // Vérifier si l'email existe
        if (repository.findByEmail(dto.getEmail()).isPresent()) {
            throw new IllegalArgumentException(
                "Email " + dto.getEmail() + " already registered"
            );
        }

        // Créer l'utilisateur
        User user = User.builder()
            .name(dto.getName())
            .email(dto.getEmail())
            .age(dto.getAge())
            .status(UserStatus.ACTIVE)
            .roles(dto.getRoles() != null ? dto.getRoles() : List.of("user"))
            .createdAt(Instant.now())
            .updatedAt(Instant.now())
            .loginCount(0)
            .build();

        try {
            repository.insertOne(user);
            logger.info("New user registered: {}", user.getEmail());

            return convertToPublicDTO(user);

        } catch (MongoWriteException e) {
            if (e.getCode() == 11000) { // Duplicate key
                throw new IllegalArgumentException("Email already exists");
            }
            throw e;
        }
    }

    public User authenticateUser(String email, String password) {
        User user = repository.findByEmail(email)
            .orElseThrow(() -> new IllegalArgumentException("Invalid credentials"));

        if (user.getStatus() != UserStatus.ACTIVE) {
            throw new IllegalStateException("Account is not active");
        }

        // TODO: Vérifier le mot de passe hashé

        // Incrémenter le compteur de connexions
        repository.incrementLoginCount(user.getId());

        logger.info("User logged in: {}", user.getEmail());
        return user;
    }

    public UserPublicDTO getUserById(String userId) {
        User user = repository.findById(userId)
            .orElseThrow(() -> new IllegalArgumentException("User not found"));

        return convertToPublicDTO(user);
    }

    public UserPublicDTO updateUser(String userId, UpdateUserDTO dto) {
        Document updates = new Document();

        if (dto.getName() != null) {
            updates.append("name", dto.getName());
        }
        if (dto.getEmail() != null) {
            updates.append("email", dto.getEmail());
        }
        if (dto.getAge() != null) {
            updates.append("age", dto.getAge());
        }

        if (updates.isEmpty()) {
            throw new IllegalArgumentException("No updates provided");
        }

        boolean success = repository.updateProfile(new ObjectId(userId), updates);

        if (!success) {
            throw new IllegalStateException("Failed to update user");
        }

        return getUserById(userId);
    }

    public List<UserPublicDTO> searchUsers(String query, int limit) {
        return repository.searchUsers(query, limit).stream()
            .map(this::convertToPublicDTO)
            .collect(Collectors.toList());
    }

    public UserRepository.UserStatistics getDashboardStats() {
        return repository.getStatistics();
    }

    public boolean suspendUser(String userId) {
        Document updates = new Document("status", UserStatus.SUSPENDED);
        return repository.updateProfile(new ObjectId(userId), updates);
    }

    private UserPublicDTO convertToPublicDTO(User user) {
        return UserPublicDTO.builder()
            .id(user.getId().toHexString())
            .name(user.getName())
            .email(user.getEmail())
            .status(user.getStatus())
            .createdAt(user.getCreatedAt())
            .build();
    }
}
```

## Opérations CRUD avancées

### Recherche avec pagination

```java
// service/ProductService.java
package com.example.service;

import com.mongodb.client.model.Filters;
import com.mongodb.client.model.Sorts;
import org.bson.conversions.Bson;

import java.util.ArrayList;
import java.util.List;

public class PaginatedResult<T> {
    private final List<T> data;
    private final long total;
    private final int page;
    private final int pageSize;
    private final int totalPages;

    public PaginatedResult(List<T> data, long total, int page, int pageSize) {
        this.data = data;
        this.total = total;
        this.page = page;
        this.pageSize = pageSize;
        this.totalPages = (int) Math.ceil((double) total / pageSize);
    }

    // Getters
    public List<T> getData() { return data; }
    public long getTotal() { return total; }
    public int getPage() { return page; }
    public int getPageSize() { return pageSize; }
    public int getTotalPages() { return totalPages; }
}

public class ProductService {

    public PaginatedResult<Product> searchProducts(
        String category,
        Double minPrice,
        Double maxPrice,
        List<String> tags,
        String searchTerm,
        int page,
        int pageSize
    ) {
        List<Bson> filters = new ArrayList<>();

        // Construction des filtres
        if (category != null) {
            filters.add(Filters.eq("category", category));
        }

        if (minPrice != null || maxPrice != null) {
            if (minPrice != null && maxPrice != null) {
                filters.add(Filters.and(
                    Filters.gte("price.amount", minPrice),
                    Filters.lte("price.amount", maxPrice)
                ));
            } else if (minPrice != null) {
                filters.add(Filters.gte("price.amount", minPrice));
            } else {
                filters.add(Filters.lte("price.amount", maxPrice));
            }
        }

        if (tags != null && !tags.isEmpty()) {
            filters.add(Filters.all("tags", tags));
        }

        if (searchTerm != null && !searchTerm.isEmpty()) {
            filters.add(Filters.or(
                Filters.regex("name", searchTerm, "i"),
                Filters.regex("description", searchTerm, "i")
            ));
        }

        Bson finalFilter = filters.isEmpty()
            ? new Document()
            : Filters.and(filters);

        // Compter le total
        long total = collection.countDocuments(finalFilter);

        // Récupérer les données
        int skip = (page - 1) * pageSize;
        List<Product> products = collection.find(finalFilter)
            .sort(Sorts.descending("createdAt"))
            .skip(skip)
            .limit(pageSize)
            .into(new ArrayList<>());

        return new PaginatedResult<>(products, total, page, pageSize);
    }
}
```

### Bulk Operations

```java
// Mise à jour en masse
import com.mongodb.client.model.UpdateOneModel;
import com.mongodb.client.model.WriteModel;
import com.mongodb.bulk.BulkWriteResult;

public class ProductService {

    public BulkUpdateResult bulkUpdatePrices(List<PriceUpdate> updates) {
        List<WriteModel<Product>> operations = new ArrayList<>();

        for (PriceUpdate update : updates) {
            operations.add(new UpdateOneModel<>(
                Filters.eq("_id", new ObjectId(update.getProductId())),
                Updates.combine(
                    Updates.set("price.amount", update.getNewPrice()),
                    Updates.set("updatedAt", Instant.now())
                )
            ));
        }

        BulkWriteResult result = collection.bulkWrite(
            operations,
            new BulkWriteOptions().ordered(false)
        );

        return new BulkUpdateResult(
            result.getMatchedCount(),
            result.getModifiedCount()
        );
    }

    public static class BulkUpdateResult {
        private final int matched;
        private final int modified;

        public BulkUpdateResult(int matched, int modified) {
            this.matched = matched;
            this.modified = modified;
        }

        public int getMatched() { return matched; }
        public int getModified() { return modified; }
    }
}
```

## Agrégations complexes

```java
// service/AnalyticsService.java
package com.example.service;

import com.mongodb.client.AggregateIterable;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.model.*;
import org.bson.Document;
import org.bson.conversions.Bson;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class AnalyticsService {

    private final MongoCollection<Document> ordersCollection;

    public SalesAnalytics getSalesAnalytics(Instant startDate, Instant endDate) {
        List<Bson> pipeline = Arrays.asList(
            // Filtrer par période
            Aggregates.match(Filters.and(
                Filters.gte("createdAt", startDate),
                Filters.lte("createdAt", endDate),
                Filters.eq("status", "completed")
            )),

            // Dérouler les items
            Aggregates.unwind("$items"),

            // Jointure avec products
            Aggregates.lookup(
                "products",
                "items.productId",
                "_id",
                "productDetails"
            ),

            Aggregates.unwind("$productDetails"),

            // Grouper par catégorie
            Aggregates.group(
                "$productDetails.category",
                Accumulators.sum("totalSales",
                    new Document("$multiply", Arrays.asList(
                        "$items.quantity",
                        "$items.price"
                    ))
                ),
                Accumulators.sum("totalItems", "$items.quantity"),
                Accumulators.addToSet("uniqueProducts", "$items.productId"),
                Accumulators.avg("avgOrderValue", "$totalAmount")
            ),

            // Ajouter le nombre de produits uniques
            Aggregates.addFields(
                new Field<>("uniqueProductsCount",
                    new Document("$size", "$uniqueProducts"))
            ),

            // Projeter
            Aggregates.project(Projections.fields(
                Projections.computed("category", "$_id"),
                Projections.include("totalSales", "totalItems", "uniqueProductsCount"),
                Projections.computed("avgOrderValue",
                    new Document("$round", Arrays.asList("$avgOrderValue", 2))),
                Projections.excludeId()
            )),

            // Trier
            Aggregates.sort(Sorts.descending("totalSales"))
        );

        List<Document> results = ordersCollection.aggregate(pipeline)
            .into(new ArrayList<>());

        return new SalesAnalytics(startDate, endDate, results);
    }

    public ProductDashboard getProductDashboard() {
        List<Bson> pipeline = Arrays.asList(
            Aggregates.facet(
                // Statistiques générales
                new Facet("generalStats",
                    Aggregates.group(null,
                        Accumulators.sum("totalProducts", 1),
                        Accumulators.avg("avgPrice", "$price.amount"),
                        Accumulators.sum("totalInventory", "$inventory.quantity"),
                        Accumulators.sum("outOfStock",
                            new Document("$cond", Arrays.asList(
                                new Document("$eq", Arrays.asList("$inventory.quantity", 0)),
                                1,
                                0
                            ))
                        )
                    )
                ),

                // Par catégorie
                new Facet("byCategory",
                    Aggregates.group("$category",
                        Accumulators.sum("count", 1),
                        Accumulators.avg("avgPrice", "$price.amount")
                    ),
                    Aggregates.sort(Sorts.descending("count"))
                ),

                // Distribution des prix
                new Facet("priceDistribution",
                    Aggregates.bucket(
                        "$price.amount",
                        Arrays.asList(0.0, 10.0, 50.0, 100.0, 500.0, 1000.0),
                        new BucketOptions()
                            .defaultBucket("1000+")
                            .output(Accumulators.sum("count", 1))
                    )
                ),

                // Top produits
                new Facet("topProducts",
                    Aggregates.sort(Sorts.descending("reviews.rating")),
                    Aggregates.limit(10),
                    Aggregates.project(Projections.fields(
                        Projections.include("name"),
                        Projections.computed("price", "$price.amount"),
                        Projections.computed("rating", "$reviews.rating")
                    ))
                )
            )
        );

        Document result = productsCollection.aggregate(pipeline)
            .first();

        return new ProductDashboard(result);
    }

    public static class SalesAnalytics {
        private final Instant startDate;
        private final Instant endDate;
        private final List<Document> byCategory;

        public SalesAnalytics(Instant startDate, Instant endDate, List<Document> byCategory) {
            this.startDate = startDate;
            this.endDate = endDate;
            this.byCategory = byCategory;
        }

        // Getters
    }

    public static class ProductDashboard {
        private final Document data;

        public ProductDashboard(Document data) {
            this.data = data;
        }

        public Document getData() { return data; }
    }
}
```

## Transactions

```java
// service/TransactionService.java
package com.example.service;

import com.mongodb.TransactionOptions;
import com.mongodb.WriteConcern;
import com.mongodb.ReadConcern;
import com.mongodb.ReadPreference;
import com.mongodb.client.ClientSession;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.model.Filters;
import com.mongodb.client.model.Updates;
import org.bson.Document;
import org.bson.types.ObjectId;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;

public class TransactionService {

    private static final Logger logger = LoggerFactory.getLogger(TransactionService.class);
    private final MongoClient client;

    public TransactionService(MongoClient client) {
        this.client = client;
    }

    public void transferFunds(
        ObjectId fromAccountId,
        ObjectId toAccountId,
        double amount
    ) {
        // Configuration de la transaction
        TransactionOptions txnOptions = TransactionOptions.builder()
            .readPreference(ReadPreference.primary())
            .readConcern(ReadConcern.SNAPSHOT)
            .writeConcern(WriteConcern.MAJORITY)
            .build();

        try (ClientSession session = client.startSession()) {
            session.startTransaction(txnOptions);

            try {
                MongoCollection<Document> accounts =
                    client.getDatabase("myapp").getCollection("accounts");

                // Débiter le compte source
                Document debitResult = accounts.findOneAndUpdate(
                    session,
                    Filters.and(
                        Filters.eq("_id", fromAccountId),
                        Filters.gte("balance", amount)
                    ),
                    Updates.combine(
                        Updates.inc("balance", -amount),
                        Updates.push("transactions", new Document()
                            .append("type", "debit")
                            .append("amount", amount)
                            .append("timestamp", Instant.now())
                        )
                    )
                );

                if (debitResult == null) {
                    throw new IllegalStateException("Insufficient funds or account not found");
                }

                // Créditer le compte destination
                accounts.updateOne(
                    session,
                    Filters.eq("_id", toAccountId),
                    Updates.combine(
                        Updates.inc("balance", amount),
                        Updates.push("transactions", new Document()
                            .append("type", "credit")
                            .append("amount", amount)
                            .append("timestamp", Instant.now())
                        )
                    )
                );

                // Commit
                session.commitTransaction();
                logger.info("✅ Transaction completed successfully");

            } catch (Exception e) {
                // Rollback automatique
                session.abortTransaction();
                logger.error("❌ Transaction failed: {}", e.getMessage());
                throw e;
            }
        }
    }

    public OrderResult createOrderWithInventoryUpdate(OrderData orderData) {
        TransactionOptions txnOptions = TransactionOptions.builder()
            .readPreference(ReadPreference.primary())
            .readConcern(ReadConcern.SNAPSHOT)
            .writeConcern(WriteConcern.MAJORITY)
            .build();

        int maxRetries = 3;
        Exception lastException = null;

        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try (ClientSession session = client.startSession()) {
                session.startTransaction(txnOptions);

                try {
                    MongoCollection<Document> orders =
                        client.getDatabase("myapp").getCollection("orders");
                    MongoCollection<Document> products =
                        client.getDatabase("myapp").getCollection("products");
                    MongoCollection<Document> users =
                        client.getDatabase("myapp").getCollection("users");

                    // 1. Réserver l'inventaire
                    for (OrderItem item : orderData.getItems()) {
                        Document updateResult = products.findOneAndUpdate(
                            session,
                            Filters.and(
                                Filters.eq("_id", item.getProductId()),
                                Filters.gte("inventory.quantity", item.getQuantity())
                            ),
                            Updates.combine(
                                Updates.inc("inventory.quantity", -item.getQuantity()),
                                Updates.set("updatedAt", Instant.now())
                            )
                        );

                        if (updateResult == null) {
                            throw new IllegalStateException(
                                "Product " + item.getProductId() + " out of stock"
                            );
                        }
                    }

                    // 2. Créer la commande
                    Document order = new Document()
                        .append("userId", orderData.getUserId())
                        .append("items", orderData.getItemsAsDocuments())
                        .append("status", "pending")
                        .append("totalAmount", orderData.getTotalAmount())
                        .append("createdAt", Instant.now())
                        .append("updatedAt", Instant.now());

                    orders.insertOne(session, order);
                    ObjectId orderId = order.getObjectId("_id");

                    // 3. Mettre à jour les stats utilisateur
                    users.updateOne(
                        session,
                        Filters.eq("_id", orderData.getUserId()),
                        Updates.combine(
                            Updates.inc("stats.totalOrders", 1),
                            Updates.set("stats.lastOrderDate", Instant.now())
                        )
                    );

                    // Commit
                    session.commitTransaction();
                    logger.info("✅ Order created: {}", orderId);

                    return new OrderResult(orderId, true);

                } catch (Exception e) {
                    session.abortTransaction();
                    lastException = e;

                    // Retry sur les erreurs transitoires
                    if (e.getMessage().contains("TransientTransactionError")
                        && attempt < maxRetries) {
                        logger.warn("Transaction failed (attempt {}), retrying...", attempt);
                        Thread.sleep(100 * attempt);
                        continue;
                    }

                    throw e;
                }
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Transaction interrupted", ie);
            }
        }

        throw new RuntimeException(
            "Transaction failed after " + maxRetries + " attempts",
            lastException
        );
    }

    public static class OrderResult {
        private final ObjectId orderId;
        private final boolean success;

        public OrderResult(ObjectId orderId, boolean success) {
            this.orderId = orderId;
            this.success = success;
        }

        public ObjectId getOrderId() { return orderId; }
        public boolean isSuccess() { return success; }
    }
}
```

## Driver Reactive Streams

```java
// Pour les applications reactive (WebFlux, etc.)
package com.example.reactive;

import com.mongodb.reactivestreams.client.MongoClient;
import com.mongodb.reactivestreams.client.MongoClients;
import com.mongodb.reactivestreams.client.MongoCollection;
import com.mongodb.reactivestreams.client.MongoDatabase;
import com.example.models.User;
import org.reactivestreams.Publisher;
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public class ReactiveUserRepository {

    private final MongoCollection<User> collection;

    public ReactiveUserRepository(MongoClient client) {
        MongoDatabase database = client.getDatabase("myapp");
        this.collection = database.getCollection("users", User.class);
    }

    public Mono<User> findById(String id) {
        return Mono.from(
            collection.find(Filters.eq("_id", new ObjectId(id)))
                .first()
        );
    }

    public Flux<User> findAll() {
        return Flux.from(collection.find());
    }

    public Mono<User> save(User user) {
        return Mono.from(collection.insertOne(user))
            .map(result -> {
                user.setId(result.getInsertedId().asObjectId().getValue());
                return user;
            });
    }

    public Mono<Long> count() {
        return Mono.from(collection.countDocuments());
    }
}
```

## Spring Data MongoDB

### Configuration Spring Boot

```java
// config/SpringMongoConfig.java
package com.example.config;

import com.mongodb.ConnectionString;
import com.mongodb.MongoClientSettings;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.config.AbstractMongoClientConfiguration;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;

import java.util.concurrent.TimeUnit;

@Configuration
@EnableMongoRepositories(basePackages = "com.example.repository")
public class SpringMongoConfig extends AbstractMongoClientConfiguration {

    @Override
    protected String getDatabaseName() {
        return "myapp";
    }

    @Override
    public MongoClient mongoClient() {
        ConnectionString connectionString = new ConnectionString(
            "mongodb://localhost:27017"
        );

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .applyToConnectionPoolSettings(builder -> builder
                .maxSize(100)
                .minSize(10)
                .maxConnectionIdleTime(60, TimeUnit.SECONDS)
            )
            .retryWrites(true)
            .retryReads(true)
            .build();

        return MongoClients.create(settings);
    }

    @Bean
    public MongoTemplate mongoTemplate() {
        return new MongoTemplate(mongoClient(), getDatabaseName());
    }
}
```

### Entité Spring Data

```java
// models/UserEntity.java
package com.example.models;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

import jakarta.validation.constraints.*;
import java.time.Instant;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "users")
public class UserEntity {

    @Id
    private String id;

    @NotBlank
    @Field("name")
    private String name;

    @NotBlank
    @Email
    @Indexed(unique = true)
    @Field("email")
    private String email;

    @Min(18)
    @Max(120)
    @Field("age")
    private Integer age;

    @NotNull
    @Builder.Default
    @Indexed
    @Field("status")
    private UserStatus status = UserStatus.ACTIVE;

    @NotNull
    @Builder.Default
    @Field("roles")
    private List<String> roles = List.of("user");

    @Field("preferences")
    private UserPreferences preferences;

    @Builder.Default
    @Field("created_at")
    private Instant createdAt = Instant.now();

    @Builder.Default
    @Field("updated_at")
    private Instant updatedAt = Instant.now();

    @Field("last_login")
    private Instant lastLogin;

    @Builder.Default
    @Field("login_count")
    private Integer loginCount = 0;
}
```

### Repository Spring Data

```java
// repository/SpringUserRepository.java
package com.example.repository;

import com.example.models.UserEntity;
import com.example.models.UserStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

@Repository
public interface SpringUserRepository extends MongoRepository<UserEntity, String> {

    Optional<UserEntity> findByEmail(String email);

    List<UserEntity> findByStatus(UserStatus status);

    Page<UserEntity> findByStatusOrderByCreatedAtDesc(
        UserStatus status,
        Pageable pageable
    );

    List<UserEntity> findByRolesContaining(String role);

    Long countByStatus(UserStatus status);

    Boolean existsByEmail(String email);

    @Query("{'createdAt': {$gte: ?0, $lte: ?1}}")
    List<UserEntity> findByCreatedAtBetween(Instant start, Instant end);

    @Query("{'$or': [{'name': {$regex: ?0, $options: 'i'}}, {'email': {$regex: ?0, $options: 'i'}}]}")
    List<UserEntity> searchByNameOrEmail(String searchTerm);
}
```

### Service Spring

```java
// service/SpringUserService.java
package com.example.service;

import com.example.dto.CreateUserDTO;
import com.example.dto.UserPublicDTO;
import com.example.models.UserEntity;
import com.example.models.UserStatus;
import com.example.repository.SpringUserRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Service
public class SpringUserService {

    private final SpringUserRepository repository;

    public SpringUserService(SpringUserRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public UserPublicDTO registerUser(CreateUserDTO dto) {
        if (repository.existsByEmail(dto.getEmail())) {
            throw new IllegalArgumentException("Email already exists");
        }

        UserEntity user = UserEntity.builder()
            .name(dto.getName())
            .email(dto.getEmail())
            .age(dto.getAge())
            .status(UserStatus.ACTIVE)
            .roles(dto.getRoles())
            .build();

        user = repository.save(user);

        return convertToDTO(user);
    }

    public Page<UserPublicDTO> getActiveUsers(int page, int size) {
        PageRequest pageRequest = PageRequest.of(
            page,
            size,
            Sort.by(Sort.Direction.DESC, "createdAt")
        );

        return repository.findByStatusOrderByCreatedAtDesc(
            UserStatus.ACTIVE,
            pageRequest
        ).map(this::convertToDTO);
    }

    private UserPublicDTO convertToDTO(UserEntity user) {
        return UserPublicDTO.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .status(user.getStatus())
            .createdAt(user.getCreatedAt())
            .build();
    }
}
```

## Gestion d'erreurs

```java
// exception/MongoDBExceptionHandler.java
package com.example.exception;

import com.mongodb.MongoException;
import com.mongodb.MongoWriteException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MongoDBExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(MongoDBExceptionHandler.class);

    public static void handle(Exception e) {
        if (e instanceof MongoWriteException) {
            MongoWriteException mwe = (MongoWriteException) e;

            if (mwe.getCode() == 11000) {
                // Duplicate key error
                logger.error("Duplicate key error: {}", mwe.getMessage());
                throw new IllegalArgumentException("Duplicate key error");
            }
        }

        if (e instanceof MongoException) {
            MongoException me = (MongoException) e;

            if (me.getCode() == 50) {
                // MaxTimeMSExpired
                logger.error("Query timeout exceeded");
                throw new RuntimeException("Query timeout exceeded");
            }

            if (me.getCode() == 13) {
                // Unauthorized
                logger.error("Authentication failed");
                throw new RuntimeException("Authentication failed");
            }
        }

        logger.error("Unexpected MongoDB error", e);
        throw new RuntimeException("Database error occurred", e);
    }
}
```

## Bonnes pratiques

### ✅ DO - À faire

```java
// 1. Utiliser le singleton pour le client
MongoClient client = MongoDBConnection.getInstance().getClient();

// 2. Utiliser POJOs avec Lombok
@Data
@Builder
public class User {
    private ObjectId id;
    private String name;
}

// 3. Typer les collections
MongoCollection<User> users = db.getCollection("users", User.class);

// 4. Gérer les index au démarrage
collection.createIndex(Indexes.ascending("email"),
    new IndexOptions().unique(true));

// 5. Utiliser le repository pattern
UserRepository userRepo = new UserRepository();

// 6. Fermer les ressources
try (MongoClient client = MongoClients.create(uri)) {
    // Opérations
}

// 7. Utiliser des projections
collection.find()
    .projection(Projections.exclude("password"))
    .into(new ArrayList<>());

// 8. Gérer les erreurs spécifiques
try {
    collection.insertOne(doc);
} catch (MongoWriteException e) {
    if (e.getCode() == 11000) {
        // Handle duplicate
    }
}

// 9. Utiliser Spring Data pour simplifier
@Repository
interface UserRepository extends MongoRepository<User, String> {
    Optional<User> findByEmail(String email);
}

// 10. Logger les opérations importantes
logger.info("User created: {}", user.getEmail());
```

### ❌ DON'T - À éviter

```java
// ❌ Créer un nouveau client par requête
// ❌ Ne pas typer les collections (utiliser Document partout)
// ❌ Ignorer les erreurs
// ❌ Ne pas fermer les ressources
// ❌ Ne pas utiliser d'index
// ❌ Faire des requêtes N+1
// ❌ Charger tous les documents en mémoire
// ❌ Ne pas utiliser de DTO pour les APIs
```

## Conclusion

Le driver Java MongoDB offre une API robuste et performante :

1. **Driver synchrone** : Production-ready, performant, typé
2. **Driver reactive** : Pour applications WebFlux
3. **Spring Data** : Simplifie énormément le code
4. **POJOs** : Mapping objet transparent
5. **Transactions** : Support ACID complet
6. **Performance** : Parmi les meilleurs drivers

**Points clés** :
- Singleton pour le client
- POJO mapping avec codecs
- Repository pattern
- Gestion d'erreurs robuste
- Spring Data pour productivité
- Index appropriés

---


⏭️ [Driver C# / .NET](/15-drivers-integration-applicative/05-driver-csharp-dotnet.md)
