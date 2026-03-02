# Database Knowledge Base — SQL Server 2025 + MongoDB Atlas
_Compiled: 2026-02-28_

---

## Decision Matrix: When to Use Which

| Scenario | Use |
|---|---|
| Transactional data, complex relations, strict consistency | **SQL Server** |
| Audit trails, financial ledgers, reporting | **SQL Server** |
| Flexible/evolving schema, hierarchical data, rapid iteration | **MongoDB** |
| Free-text search, autocomplete, faceting, relevance ranking | **MongoDB Atlas Search** |
| Semantic / AI similarity search, RAG, embeddings | **MongoDB Atlas Vector Search** |
| IoT / time-series data | **MongoDB Time Series Collections** |
| Geospatial queries | **MongoDB** (built-in geo support) |

---

## PART 1: SQL SERVER 2025

### What's New in SQL Server 2025 (v17.x) — Key Enterprise Features

#### AI & Vector
- **`vector` data type** — native storage of vector embeddings (float32 or float16), JSON-array exposed
- **Vector indexes** — approximate nearest neighbor (ANN) search for similarity queries
- **Vector scalar functions** — compute cosine similarity, dot product, etc. in T-SQL
- **External AI models** — `CREATE EXTERNAL MODEL` to call REST AI endpoints (Azure OpenAI, etc.) from SQL
- **`sp_invoke_external_rest_endpoint`** — call REST/GraphQL/Azure Functions directly from T-SQL

#### Developer Features
- **Native `json` data type** — binary JSON (faster than `nvarchar` JSON), queryable with standard T-SQL
- **Regular expressions** — `REGEXP_LIKE`, `REGEXP_REPLACE`, `REGEXP_SUBSTR` in T-SQL (finally!)
- **Fuzzy string matching** — `SIMILARITY()`, `EDITDISTANCE()` functions
- **Change Event Streaming** — CDC-like feature publishing inserts/updates/deletes to **Azure Event Hubs** as CloudEvents (Avro or JSON) in near real-time

#### Security & Availability
- **Always Encrypted** enhancements
- **Ledger tables** — blockchain-style tamper-evident tables with cryptographic proof
- **Accelerated Database Recovery (ADR)** improvements
- **TDS 8.0** — encrypted connections by default

#### Standard Edition Limits (2025)
- Up to **4 sockets or 32 cores**
- Up to **256 GB** buffer pool memory
- Resource Governor now included (was Enterprise-only)

---

### EF Core 10 with SQL Server — Enterprise Pattern

#### NuGet Packages
```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="10.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite" Version="10.*" /> <!-- spatial -->
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.*" PrivateAssets="all" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="10.*" PrivateAssets="all" />
```

#### DbContext Configuration (Enterprise)
```csharp
// Infrastructure/Persistence/AppDbContext.cs
public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all entity configurations from assembly (clean separation)
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

        // Global query filter — soft delete
        modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
    }

    protected override void ConfigureConventions(ModelConfigurationBuilder configBuilder)
    {
        // All string props default to 256 chars max (prevents nvarchar(max))
        configBuilder.Properties<string>().HaveMaxLength(256);

        // Decimal precision convention
        configBuilder.Properties<decimal>().HavePrecision(18, 4);
    }
}
```

#### Registration in Program.cs
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.MigrationsAssembly("MyApi.Infrastructure");
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        });

    // Performance diagnostics
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
        options.LogTo(Console.WriteLine, LogLevel.Information);
    }
});
```

#### Entity Configuration (IEntityTypeConfiguration<T>)
```csharp
// Use separate config classes — NOT OnModelCreating data annotations sprawl
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.ToTable("Users", schema: "identity");

        builder.HasKey(u => u.Id);

        builder.Property(u => u.Id)
               .HasDefaultValueSql("NEWSEQUENTIALID()"); // sequential GUID for index perf

        builder.Property(u => u.Email)
               .IsRequired()
               .HasMaxLength(320);

        builder.HasIndex(u => u.Email)
               .IsUnique()
               .HasFilter("[IsDeleted] = 0"); // partial index — only active users

        builder.Property(u => u.CreatedAt)
               .HasDefaultValueSql("SYSUTCDATETIME()");

        builder.Property(u => u.RowVersion)
               .IsRowVersion(); // optimistic concurrency

        // Owned type (value object)
        builder.OwnsOne(u => u.Address, address =>
        {
            address.Property(a => a.Street).HasMaxLength(200);
            address.Property(a => a.City).HasMaxLength(100);
            address.Property(a => a.PostalCode).HasMaxLength(20);
        });

        // Navigation
        builder.HasMany(u => u.Orders)
               .WithOne(o => o.User)
               .HasForeignKey(o => o.UserId)
               .OnDelete(DeleteBehavior.Restrict); // never cascade delete in enterprise
    }
}
```

#### Domain Entities (Best Practice)
```csharp
public sealed class User : BaseEntity
{
    // Private constructor — use static factory
    private User() { }

    public static User Create(string name, string email)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentException.ThrowIfNullOrWhiteSpace(email);

        return new User
        {
            Id = Guid.NewGuid(),
            Name = name,
            Email = email.ToLowerInvariant(),
            CreatedAt = DateTimeOffset.UtcNow,
            IsActive = true
        };
    }

    public Guid Id { get; private set; }
    public string Name { get; private set; } = default!;
    public string Email { get; private set; } = default!;
    public Address Address { get; private set; } = default!;
    public bool IsActive { get; private set; }
    public bool IsDeleted { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public byte[] RowVersion { get; private set; } = default!;

    private readonly List<Order> _orders = [];
    public IReadOnlyCollection<Order> Orders => _orders.AsReadOnly();

    public void Deactivate() => IsActive = false;
    public void SoftDelete() => IsDeleted = true;
}

public abstract class BaseEntity
{
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    protected void RaiseDomainEvent(IDomainEvent @event) => _domainEvents.Add(@event);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

#### Repository Pattern
```csharp
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<User?> GetByEmailAsync(string email, CancellationToken ct = default);
    Task<PagedResult<User>> GetPagedAsync(int page, int pageSize, CancellationToken ct = default);
    Task AddAsync(User user, CancellationToken ct = default);
    void Update(User user);
    void Remove(User user);
}

public class UserRepository(AppDbContext context) : IUserRepository
{
    public async Task<User?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        await context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, ct);

    public async Task<PagedResult<User>> GetPagedAsync(
        int page, int pageSize, CancellationToken ct = default)
    {
        var query = context.Users.AsNoTracking().OrderBy(u => u.CreatedAt);

        var totalCount = await query.CountAsync(ct);
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return new PagedResult<User>(items, totalCount, page, pageSize);
    }

    public async Task AddAsync(User user, CancellationToken ct = default) =>
        await context.Users.AddAsync(user, ct);

    public void Update(User user) => context.Users.Update(user);
    public void Remove(User user) => context.Users.Remove(user);
}
```

#### Unit of Work
```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork(AppDbContext context, IPublisher publisher) : IUnitOfWork
{
    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Dispatch domain events before saving
        var entities = context.ChangeTracker.Entries<BaseEntity>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var domainEvents = entities.SelectMany(e => e.DomainEvents).ToList();
        entities.ForEach(e => e.ClearDomainEvents());

        var result = await context.SaveChangesAsync(ct);

        foreach (var @event in domainEvents)
            await publisher.Publish(@event, ct);

        return result;
    }
}
```

#### EF Core Migrations (Enterprise)
```bash
# Never run migrations automatically in production
# Always generate SQL script and review before applying

# Add migration
dotnet ef migrations add AddUserAddressTable \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Generate idempotent SQL script (for prod deployments)
dotnet ef migrations script \
  --idempotent \
  --output migrations.sql \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Or apply via pipeline (CI/CD)
dotnet ef database update \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api
```

#### Advanced Queries
```csharp
// Compiled query (cached execution plan — best for hot paths)
private static readonly Func<AppDbContext, Guid, Task<User?>> GetUserByIdCompiled =
    EF.CompileAsyncQuery((AppDbContext ctx, Guid id) =>
        ctx.Users.FirstOrDefault(u => u.Id == id));

// Raw SQL for complex queries (parameterized — no injection risk)
var users = await context.Users
    .FromSqlRaw("SELECT * FROM identity.Users WHERE Email LIKE {0}", $"%{domain}%")
    .AsNoTracking()
    .ToListAsync(ct);

// Bulk operations (EF Core 10 native)
await context.Users
    .Where(u => u.IsDeleted && u.CreatedAt < cutoff)
    .ExecuteDeleteAsync(ct);  // single SQL DELETE, no tracking

await context.Users
    .Where(u => !u.IsActive)
    .ExecuteUpdateAsync(s =>
        s.SetProperty(u => u.IsDeleted, true)
         .SetProperty(u => u.DeletedAt, DateTimeOffset.UtcNow), ct);

// Named query filters (EF Core 10) — disable global filter selectively
var allUsers = await context.Users
    .IgnoreQueryFilters()  // includes soft-deleted
    .ToListAsync(ct);
```

#### SQL Server 2025 — Native JSON Data Type
```csharp
// Entity with native JSON column
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = default!;
    public JsonDocument Metadata { get; set; } = default!;  // native JSON type
}

// EF Core config
builder.Property(p => p.Metadata).HasColumnType("json");

// Query JSON fields
var products = await context.Products
    .Where(p => EF.Functions.JsonValue(p.Metadata, "$.category") == "electronics")
    .ToListAsync(ct);
```

#### Connection String (Enterprise Format)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;Application Name=MyApi"
  }
}
```

**Note:** Always use **Managed Identity** for Azure SQL — no passwords in connection strings.

---

## PART 2: MONGODB

### Architecture Concepts

| Concept | Description |
|---|---|
| **Document** | JSON-like record (BSON internally). Flexible schema. |
| **Collection** | Group of documents — analogous to SQL table, but schemaless. |
| **Database** | Contains collections |
| **`_id`** | Primary key — `ObjectId` by default (12-byte, globally unique) |
| **Replica Set** | 3+ nodes, automatic failover, primary + secondaries |
| **Sharding** | Horizontal partitioning across nodes by shard key |
| **BSON** | Binary JSON — supports dates, ObjectId, binary data, decimals |
| **Transactions** | Full ACID multi-document transactions (since v4.0) |

### .NET Driver Setup

#### NuGet Packages
```xml
<PackageReference Include="MongoDB.Driver" Version="3.*" />
<PackageReference Include="MongoDB.Driver.Core" Version="3.*" />
<PackageReference Include="MongoDB.Bson" Version="3.*" />
```

#### Registration in Program.cs (Enterprise)
```csharp
// Configuration class
public record MongoDbOptions
{
    public string ConnectionString { get; init; } = default!;
    public string DatabaseName { get; init; } = default!;
}

// Register client as singleton (thread-safe, reuse connections)
builder.Services.Configure<MongoDbOptions>(
    builder.Configuration.GetSection("MongoDb"));

builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var options = sp.GetRequiredService<IOptions<MongoDbOptions>>().Value;

    var settings = MongoClientSettings.FromConnectionString(options.ConnectionString);
    settings.ServerApi = new ServerApi(ServerApiVersion.V1);  // stable API
    settings.RetryWrites = true;
    settings.RetryReads = true;
    settings.ConnectTimeout = TimeSpan.FromSeconds(10);
    settings.SocketTimeout = TimeSpan.FromSeconds(30);
    settings.ServerSelectionTimeout = TimeSpan.FromSeconds(10);

    // OpenTelemetry instrumentation
    settings.ClusterConfigurator = cb =>
        cb.Subscribe<CommandStartedEvent>(e =>
            Log.Debug("MongoDB: {Command}", e.CommandName));

    return new MongoClient(settings);
});

builder.Services.AddSingleton<IMongoDatabase>(sp =>
{
    var options = sp.GetRequiredService<IOptions<MongoDbOptions>>().Value;
    var client = sp.GetRequiredService<IMongoClient>();
    return client.GetDatabase(options.DatabaseName);
});
```

#### Document Model (C# Conventions)
```csharp
// BSON annotations control serialization
[BsonCollection("products")]  // custom attribute for collection name
public class Product
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string Id { get; set; } = ObjectId.GenerateNewId().ToString();

    [BsonElement("name")]
    public string Name { get; set; } = default!;

    [BsonElement("price")]
    [BsonRepresentation(BsonType.Decimal128)]
    public decimal Price { get; set; }

    [BsonElement("tags")]
    public List<string> Tags { get; set; } = [];

    [BsonElement("specifications")]
    public ProductSpecifications Specifications { get; set; } = default!;

    [BsonElement("createdAt")]
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;

    [BsonElement("updatedAt")]
    [BsonIgnoreIfNull]
    public DateTimeOffset? UpdatedAt { get; set; }

    [BsonIgnoreIfDefault]
    public bool IsDeleted { get; set; }
}

public class ProductSpecifications
{
    [BsonElement("weight")]
    public double WeightKg { get; set; }

    [BsonElement("dimensions")]
    public Dimensions Dimensions { get; set; } = default!;

    [BsonElement("color")]
    public string Color { get; set; } = default!;
}
```

#### Repository Pattern for MongoDB
```csharp
public interface IMongoRepository<T>
{
    Task<T?> GetByIdAsync(string id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task<PagedResult<T>> GetPagedAsync(FilterDefinition<T> filter, int page, int pageSize, CancellationToken ct = default);
    Task InsertAsync(T document, CancellationToken ct = default);
    Task InsertManyAsync(IEnumerable<T> documents, CancellationToken ct = default);
    Task<bool> ReplaceAsync(string id, T document, CancellationToken ct = default);
    Task<bool> UpdateAsync(string id, UpdateDefinition<T> update, CancellationToken ct = default);
    Task<bool> DeleteAsync(string id, CancellationToken ct = default);
    Task<long> CountAsync(FilterDefinition<T> filter, CancellationToken ct = default);
}

public class MongoRepository<T>(IMongoDatabase database) : IMongoRepository<T>
    where T : class
{
    private readonly IMongoCollection<T> _collection =
        database.GetCollection<T>(GetCollectionName(typeof(T)));

    private static string GetCollectionName(Type type) =>
        type.GetCustomAttribute<BsonCollectionAttribute>()?.CollectionName
            ?? type.Name.ToLowerInvariant() + "s";

    public async Task<T?> GetByIdAsync(string id, CancellationToken ct = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        return await _collection.Find(filter).FirstOrDefaultAsync(ct);
    }

    public async Task InsertAsync(T document, CancellationToken ct = default)
    {
        await _collection.InsertOneAsync(document,
            new InsertOneOptions { BypassDocumentValidation = false }, ct);
    }

    public async Task InsertManyAsync(IEnumerable<T> documents, CancellationToken ct = default)
    {
        await _collection.InsertManyAsync(documents,
            new InsertManyOptions { IsOrdered = false }, ct); // IsOrdered=false = parallel inserts
    }

    public async Task<bool> ReplaceAsync(string id, T document, CancellationToken ct = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        var result = await _collection.ReplaceOneAsync(filter, document,
            new ReplaceOptions { IsUpsert = false }, ct);
        return result.ModifiedCount > 0;
    }

    public async Task<bool> UpdateAsync(string id, UpdateDefinition<T> update, CancellationToken ct = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        var result = await _collection.UpdateOneAsync(filter, update,
            new UpdateOptions { IsUpsert = false }, ct);
        return result.ModifiedCount > 0;
    }

    public async Task<bool> DeleteAsync(string id, CancellationToken ct = default)
    {
        var filter = Builders<T>.Filter.Eq("_id", ObjectId.Parse(id));
        var result = await _collection.DeleteOneAsync(filter, ct);
        return result.DeletedCount > 0;
    }

    public async Task<PagedResult<T>> GetPagedAsync(
        FilterDefinition<T> filter, int page, int pageSize, CancellationToken ct = default)
    {
        var countFacet = AggregateFacet.Create("count",
            PipelineDefinition<T, AggregateCountResult>.Create(new[]
            {
                PipelineStageDefinitionBuilder.Count<T>()
            }));

        var dataFacet = AggregateFacet.Create("data",
            PipelineDefinition<T, T>.Create(new[]
            {
                PipelineStageDefinitionBuilder.Skip<T>((page - 1) * pageSize),
                PipelineStageDefinitionBuilder.Limit<T>(pageSize)
            }));

        var aggregation = await _collection.Aggregate()
            .Match(filter)
            .Facet(countFacet, dataFacet)
            .ToListAsync(ct);

        var count = aggregation.First()
            .Facets.First(f => f.Name == "count")
            .Output<AggregateCountResult>()?.FirstOrDefault()?.Count ?? 0;

        var data = aggregation.First()
            .Facets.First(f => f.Name == "data")
            .Output<T>();

        return new PagedResult<T>(data, (int)count, page, pageSize);
    }
}
```

#### CRUD Operations
```csharp
var filter = Builders<Product>.Filter;
var update = Builders<Product>.Update;
var sort = Builders<Product>.Sort;

// Find with filter
var products = await collection
    .Find(filter.And(
        filter.Eq(p => p.IsDeleted, false),
        filter.In(p => p.Tags, new[] { "electronics", "gadgets" }),
        filter.Gte(p => p.Price, 100m),
        filter.Lte(p => p.Price, 1000m)))
    .Sort(sort.Descending(p => p.CreatedAt))
    .Skip(0)
    .Limit(20)
    .ToListAsync(ct);

// Update single field
await collection.UpdateOneAsync(
    filter.Eq(p => p.Id, id),
    update.Combine(
        update.Set(p => p.Price, newPrice),
        update.Set(p => p.UpdatedAt, DateTimeOffset.UtcNow),
        update.Push(p => p.Tags, "sale")));

// Atomic find-and-modify
var updated = await collection.FindOneAndUpdateAsync(
    filter.Eq(p => p.Id, id),
    update.Inc("viewCount", 1),
    new FindOneAndUpdateOptions<Product> { ReturnDocument = ReturnDocument.After });

// Upsert
await collection.UpdateOneAsync(
    filter.Eq(p => p.Sku, sku),
    update.SetOnInsert(p => p.CreatedAt, DateTimeOffset.UtcNow)
          .Set(p => p.Price, price),
    new UpdateOptions { IsUpsert = true });
```

#### Aggregation Pipeline
```csharp
var pipeline = collection.Aggregate()
    .Match(filter.Eq(p => p.IsDeleted, false))
    .Group(p => p.Tags,
        g => new
        {
            Tag = g.Key,
            Count = g.Count(),
            AvgPrice = g.Average(p => p.Price),
            MinPrice = g.Min(p => p.Price),
            MaxPrice = g.Max(p => p.Price)
        })
    .Sort(Builders<BsonDocument>.Sort.Descending("Count"))
    .Limit(10);

var results = await pipeline.ToListAsync(ct);
```

#### Indexes (Define in Code — Run Once at Startup)
```csharp
public class ProductIndexInitializer(IMongoDatabase database)
{
    public async Task EnsureIndexesAsync(CancellationToken ct = default)
    {
        var collection = database.GetCollection<Product>("products");
        var indexKeys = Builders<Product>.IndexKeys;

        var indexes = new List<CreateIndexModel<Product>>
        {
            // Compound index for common query
            new(indexKeys.Ascending(p => p.Tags)
                          .Descending(p => p.Price),
                new CreateIndexOptions { Name = "idx_tags_price" }),

            // Unique index on SKU
            new(indexKeys.Ascending(p => p.Sku),
                new CreateIndexOptions { Unique = true, Name = "idx_sku_unique" }),

            // TTL index — auto-delete expired sessions
            new(indexKeys.Ascending("expiresAt"),
                new CreateIndexOptions
                {
                    ExpireAfter = TimeSpan.Zero,
                    Name = "idx_ttl_expires"
                }),

            // Text index for basic search (use Atlas Search for production)
            new(indexKeys.Text(p => p.Name).Text(p => p.Description),
                new CreateIndexOptions { Name = "idx_text_search" }),
        };

        await collection.Indexes.CreateManyAsync(indexes, ct);
    }
}
```

#### Multi-Document ACID Transactions
```csharp
// Use IClientSessionHandle for transactions
using var session = await client.StartSessionAsync(ct: ct);

session.StartTransaction(new TransactionOptions(
    readConcern: ReadConcern.Snapshot,
    writeConcern: WriteConcern.WMajority));

try
{
    // Both operations must succeed or both roll back
    await ordersCollection.InsertOneAsync(session, order, ct: ct);

    await inventoryCollection.UpdateOneAsync(
        session,
        Builders<Inventory>.Filter.Eq(i => i.ProductId, order.ProductId),
        Builders<Inventory>.Update.Inc(i => i.Quantity, -order.Quantity),
        ct: ct);

    await session.CommitTransactionAsync(ct);
}
catch (Exception)
{
    await session.AbortTransactionAsync(ct);
    throw;
}
```

---

## PART 3: MONGODB ATLAS SEARCH

### Architecture
- **`mongot`** — separate Lucene-based search process running alongside `mongod` on each node
- Search indexes are **separate from regular MongoDB indexes**
- Queries use the **Aggregation Pipeline** via `$search` and `$searchMeta` stages
- Indexes are eventually consistent (sync from mongod to mongot — usually < 1 second)

### Atlas Search Index Definition (JSON)
```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": [
        {
          "type": "string",
          "analyzer": "lucene.standard"
        },
        {
          "type": "autocomplete",
          "analyzer": "lucene.standard",
          "tokenization": "edgeGram",
          "minGrams": 2,
          "maxGrams": 15
        }
      ],
      "description": {
        "type": "string",
        "analyzer": "lucene.english"
      },
      "tags": {
        "type": "string",
        "analyzer": "lucene.keyword"
      },
      "price": {
        "type": "number"
      },
      "category": {
        "type": "stringFacet"
      },
      "createdAt": {
        "type": "date"
      }
    }
  }
}
```

### Atlas Search Operators Reference

| Operator | Purpose |
|---|---|
| `text` | Full-text search with analyzer |
| `phrase` | Exact phrase match |
| `autocomplete` | Prefix / partial match as-you-type |
| `compound` | Boolean logic: `must`, `mustNot`, `should`, `filter` |
| `range` | Numeric, date, or string ranges |
| `wildcard` | `*` and `?` glob patterns |
| `regex` | Regular expression matching |
| `fuzzy` | Typo-tolerant search |
| `near` | Proximity — date/number near a value |
| `moreLikeThis` | "More documents like this one" |
| `equals` | Exact field match (indexed as non-analyzed) |
| `exists` | Field presence check |
| `geoWithin` | Geospatial polygon/circle search |
| `geoShape` | Geospatial shape intersection |

### Atlas Search Queries in C# (.NET Driver)

#### 1. Basic Full-Text Search
```csharp
var results = await collection.Aggregate()
    .Search(Builders<Product>.Search.Text(
        p => p.Name,
        "wireless headphones"))
    .Project(Builders<Product>.Projection
        .Include(p => p.Name)
        .Include(p => p.Price)
        .MetaSearchScore("score"))
    .SortByDescending(p => p["score"])
    .Limit(20)
    .ToListAsync(ct);
```

#### 2. Compound Search (Must + Should + Filter)
```csharp
var searchDef = Builders<Product>.Search;

var results = await collection.Aggregate()
    .Search(searchDef.Compound()
        .Must(searchDef.Text(p => p.Name, query))           // MUST match
        .MustNot(searchDef.Equals(p => p.IsDeleted, true))  // MUST NOT be deleted
        .Should(searchDef.Text(p => p.Description, query))  // boosts score if matches
        .Filter(searchDef.Range(p => p.Price)
            .Gte(minPrice).Lte(maxPrice)))                  // filter (no score impact)
    .Project<ProductSearchResult>(
        Builders<Product>.Projection
            .Include(p => p.Id)
            .Include(p => p.Name)
            .Include(p => p.Price)
            .Include(p => p.Category)
            .MetaSearchScore("searchScore"))
    .SortByDescending(p => p["searchScore"])
    .Limit(pageSize)
    .Skip((page - 1) * pageSize)
    .ToListAsync(ct);
```

#### 3. Autocomplete (Search-as-you-type)
```csharp
// Requires field indexed as "autocomplete" type
var suggestions = await collection.Aggregate()
    .Search(Builders<Product>.Search.Autocomplete(
        p => p.Name,
        query,
        fuzzy: new SearchFuzzyOptions { MaxEdits = 1 }))
    .Limit(10)
    .Project(Builders<Product>.Projection
        .Include(p => p.Name)
        .Include(p => p.Id))
    .ToListAsync(ct);
```

#### 4. Faceted Search (Category counts + results together)
```csharp
var searchMeta = await collection.Aggregate()
    .SearchMeta(Builders<Product>.Search.Facet(
        Builders<Product>.Search.Text(p => p.Name, query),
        new SearchFacet[]
        {
            // Category facets (string)
            new StringSearchFacet("categoryFacet", p => p.Category, 10),

            // Price range facets (numeric)
            new NumberSearchFacet("priceFacet", p => p.Price,
                new BsonDocument[]
                {
                    new() { ["min"] = 0,    ["max"] = 50 },
                    new() { ["min"] = 50,   ["max"] = 100 },
                    new() { ["min"] = 100,  ["max"] = 500 },
                    new() { ["min"] = 500,  ["max"] = 1000 },
                    new() { ["min"] = 1000, ["max"] = BsonNull.Value }
                })
        }))
    .FirstAsync(ct);

// Returns counts per category and per price range
```

#### 5. Fuzzy / Typo-Tolerant Search
```csharp
var results = await collection.Aggregate()
    .Search(Builders<Product>.Search.Text(
        p => p.Name,
        query,
        new SearchFuzzyOptions
        {
            MaxEdits = 2,          // allow up to 2 character edits
            PrefixLength = 3,      // first 3 chars must match exactly
            MaxExpansions = 50     // max terms to consider
        }))
    .Limit(20)
    .ToListAsync(ct);
```

#### 6. Phrase + Wildcard Search
```csharp
// Exact phrase
var phrase = await collection.Aggregate()
    .Search(Builders<Product>.Search.Phrase(p => p.Description, "noise cancelling"))
    .ToListAsync(ct);

// Wildcard
var wildcard = await collection.Aggregate()
    .Search(Builders<Product>.Search.Wildcard(p => p.Name, "head*"))
    .ToListAsync(ct);
```

#### 7. Score Boosting (Custom Relevance)
```csharp
var results = await collection.Aggregate()
    .Search(Builders<Product>.Search.Compound()
        .Should(Builders<Product>.Search.Text(p => p.Name, query)
            .Score(s => s.Boost(3.0)))       // name matches worth 3x
        .Should(Builders<Product>.Search.Text(p => p.Description, query)
            .Score(s => s.Boost(1.0)))       // description matches worth 1x
        .Should(Builders<Product>.Search.Range(p => p.Rating)
            .Gte(4.0)
            .Score(s => s.Boost(1.5))))      // high-rated products boosted
    .ToListAsync(ct);
```

#### 8. Pagination with searchSequenceToken (Stable Paging)
```csharp
// Get first page + continuation tokens
var firstPage = await collection.Aggregate()
    .Search(searchDef,
        new SearchOptions
        {
            SearchAfter = null,  // first page
            Tracking = new SearchTrackingOptions { SearchTerms = query }
        })
    .Limit(20)
    .Project(Builders<Product>.Projection.Include(p => p.Name).SearchSequenceToken())
    .ToListAsync(ct);

// Use token for next page (stable — results won't shift)
var lastToken = firstPage.Last()["searchSequenceToken"];
var nextPage = await collection.Aggregate()
    .Search(searchDef,
        new SearchOptions { SearchAfter = lastToken })
    .Limit(20)
    .ToListAsync(ct);
```

---

## PART 4: MONGODB ATLAS VECTOR SEARCH

### What It Is
- Store vector embeddings (from OpenAI, Azure OpenAI, Cohere, etc.) as a field in MongoDB documents
- Query by **semantic similarity** using ANN (Approximate Nearest Neighbor) or ENN (Exact Nearest Neighbor)
- Combine with **Atlas Search** for hybrid search (full-text + semantic in one query)
- Power RAG (Retrieval-Augmented Generation) pipelines
- Supports up to **8192 dimensions** per vector

### Vector Search Index Definition
```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1536,
      "similarity": "cosine"
    },
    {
      "type": "filter",
      "path": "category"
    },
    {
      "type": "filter",
      "path": "price"
    }
  ]
}
```

### Similarity Functions
| Function | Best For |
|---|---|
| `cosine` | Text embeddings (most common) |
| `euclidean` | Spatial/geometric data |
| `dotProduct` | Normalized vectors (fastest) |

### Vector Search in C#
```csharp
// Document with embedding field
public class ProductWithEmbedding
{
    [BsonId]
    public string Id { get; set; } = default!;
    public string Name { get; set; } = default!;
    public string Description { get; set; } = default!;
    public float[] Embedding { get; set; } = default!;   // vector embedding
    public string Category { get; set; } = default!;
    public decimal Price { get; set; }
}

// Perform vector search
public async Task<List<ProductSearchResult>> SemanticSearchAsync(
    float[] queryEmbedding,
    string? categoryFilter = null,
    int limit = 10,
    CancellationToken ct = default)
{
    var vectorSearchStage = new BsonDocument("$vectorSearch",
        new BsonDocument
        {
            { "index", "vector_index" },
            { "path", "embedding" },
            { "queryVector", new BsonArray(queryEmbedding.Select(f => (BsonValue)(double)f)) },
            { "numCandidates", limit * 10 },   // search wider, return fewer
            { "limit", limit },
            // Pre-filter (uses indexed filter fields)
            { "filter", categoryFilter != null
                ? new BsonDocument("category", categoryFilter)
                : new BsonDocument() }
        });

    var pipeline = new BsonDocument[]
    {
        vectorSearchStage,
        new("$addFields", new BsonDocument("vectorScore",
            new BsonDocument("$meta", "vectorSearchScore"))),
        new("$project", new BsonDocument
        {
            { "name", 1 },
            { "description", 1 },
            { "category", 1 },
            { "price", 1 },
            { "vectorScore", 1 },
            { "embedding", 0 }   // exclude large embedding from results
        })
    };

    return await collection
        .Aggregate(PipelineDefinition<ProductWithEmbedding, ProductSearchResult>
            .Create(pipeline))
        .ToListAsync(ct);
}
```

### Hybrid Search (Vector + Full-Text in One Query)
```csharp
// $search stage (full-text) + $vectorSearch + $reciprocalRankFusion
var pipeline = new[]
{
    // Stage 1: Full-text search
    new BsonDocument("$search",
        new BsonDocument
        {
            { "index", "text_index" },
            { "text", new BsonDocument
                {
                    { "query", userQuery },
                    { "path", "name" }
                }
            }
        }),

    // Stage 2: Add text score
    new BsonDocument("$addFields", new BsonDocument
    {
        { "textScore", new BsonDocument("$meta", "searchScore") }
    }),

    // Union with vector search results...
    // (Hybrid search typically uses $unionWith + reciprocal rank fusion scoring)
};
```

### RAG Pattern (Retrieval-Augmented Generation)
```csharp
public class RagService(
    IMongoCollection<ProductWithEmbedding> collection,
    IEmbeddingService embeddingService,
    IAzureOpenAIService openAiService)
{
    public async Task<string> AnswerQuestionAsync(string userQuestion, CancellationToken ct)
    {
        // 1. Embed the user's question
        var queryEmbedding = await embeddingService.GetEmbeddingAsync(userQuestion, ct);

        // 2. Find semantically similar documents
        var relevantDocs = await SemanticSearchAsync(queryEmbedding, limit: 5, ct: ct);

        // 3. Build context from retrieved documents
        var context = string.Join("\n\n", relevantDocs.Select(d =>
            $"Product: {d.Name}\nDescription: {d.Description}\nPrice: ${d.Price}"));

        // 4. Ask LLM with context (RAG)
        var prompt = $"""
            Answer the following question using only the provided context.

            Context:
            {context}

            Question: {userQuestion}
            """;

        return await openAiService.CompleteAsync(prompt, ct);
    }
}
```

---

## PART 5: EF CORE + MONGODB (Alternate ORM Approach)

```xml
<!-- For teams preferring EF Core for MongoDB -->
<PackageReference Include="MongoDB.EntityFrameworkCore" Version="8.*" />
```

```csharp
// Same EF Core patterns but targeting MongoDB
public class CatalogDbContext(DbContextOptions<CatalogDbContext> options)
    : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>().ToCollection("products");
    }
}

// Registration
builder.Services.AddDbContext<CatalogDbContext>(options =>
    options.UseMongoDB(connectionString, databaseName));
```

---

## PART 6: ENTERPRISE PATTERNS — WHICH DB FOR WHAT

### SQL Server Use Cases
- User accounts, roles, permissions
- Orders, invoices, financial transactions
- Audit logs with referential integrity
- Multi-table reports with complex JOINs
- Data that requires ACID guarantees across many tables

### MongoDB Use Cases
- Product catalogs with variable attributes (electronics vs clothing have different fields)
- Content management (articles, media, comments)
- User activity/event streams
- Shopping carts (session-like, JSON-natural)
- IoT sensor data (time-series collections)
- Configuration & feature data

### Atlas Search Use Cases
- E-commerce product search (facets + filters + autocomplete)
- Document/knowledge base search
- People/company search
- Any UX where users type and expect relevant results

### Atlas Vector Search Use Cases
- "Similar products" / "You might also like"
- Semantic Q&A over internal documents
- RAG-powered chatbots
- Image similarity search
- Code semantic search

---

## PART 7: CONNECTION STRING REFERENCES

### SQL Server (Managed Identity — Enterprise)
```
Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity;Encrypt=True;TrustServerCertificate=False;
```

### SQL Server (Local Dev)
```
Server=(localdb)\mssqllocaldb;Database=mydb;Trusted_Connection=True;MultipleActiveResultSets=true;
```

### MongoDB Atlas (Standard)
```
mongodb+srv://<username>:<password>@cluster0.abc123.mongodb.net/?retryWrites=true&w=majority&appName=MyApi
```

### MongoDB Atlas (X.509 / Managed Identity — Enterprise)
```
mongodb+srv://cluster0.abc123.mongodb.net/?authMechanism=MONGODB-X509&tls=true
```

---

_Sources: learn.microsoft.com/sql-server/2025, learn.microsoft.com/ef/core, mongodb.com/docs/manual, mongodb.com/docs/atlas/atlas-search, mongodb.com/docs/atlas/atlas-vector-search_
