# ORM & ODM Knowledge Base — EF Core 10 (SQL Server) + MongoDB EF Core Provider
_Compiled: 2026-02-28_

---

## The Core Concept: ORM vs ODM

| Term | Full Name | Maps | Used With |
|---|---|---|---|
| **ORM** | Object-Relational Mapper | C# classes ↔ relational tables | SQL Server, PostgreSQL, SQLite, MySQL |
| **ODM** | Object-Document Mapper | C# classes ↔ JSON/BSON documents | MongoDB, CouchDB |

**EF Core** covers both — it has providers for relational DBs **and** MongoDB (via `MongoDB.EntityFrameworkCore`). In practice:
- Use **EF Core + SQL Server provider** for relational data
- Use **EF Core + MongoDB provider** OR **native MongoDB C# Driver** for document data
- **Native Driver** is preferred for MongoDB when you need full aggregation pipeline, Atlas Search, Vector Search — EF Core MongoDB provider is better for teams that want a unified API

---

## PART 1: EF CORE 10 — ORM FOR SQL SERVER

### Configuration Priority (highest wins)
```
Fluent API  >  Data Annotations  >  Conventions
```
Always use **Fluent API** in enterprise code — it's the most powerful and keeps entity classes clean.

---

### Three Ways to Configure a Model

#### 1. Conventions (automatic, zero code)
EF Core auto-detects:
- Property named `Id` or `{TypeName}Id` → primary key
- `string` → nullable `nvarchar(max)` (override with conventions!)
- Navigation properties → relationships
- `bool` → `bit NOT NULL`

#### 2. Data Annotations (attributes on entity classes — avoid in enterprise)
```csharp
[Table("Users", Schema = "identity")]
public class User
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }

    [Required]
    [MaxLength(100)]
    [Column("user_name")]
    public string Name { get; set; } = default!;

    [EmailAddress]
    [MaxLength(320)]
    public string Email { get; set; } = default!;
}
```
⚠️ **Avoid in enterprise** — pollutes domain entities with infrastructure concerns.

#### 3. Fluent API via IEntityTypeConfiguration<T> (enterprise standard)
```csharp
// One config class per entity — clean separation
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        // Table & Schema
        builder.ToTable("Users", schema: "identity");

        // Primary Key
        builder.HasKey(u => u.Id);
        builder.Property(u => u.Id)
               .HasDefaultValueSql("NEWSEQUENTIALID()"); // sequential GUID for SQL Server

        // Properties
        builder.Property(u => u.Name)
               .IsRequired()
               .HasMaxLength(100)
               .HasColumnName("user_name");

        builder.Property(u => u.Email)
               .IsRequired()
               .HasMaxLength(320);

        // Indexes
        builder.HasIndex(u => u.Email)
               .IsUnique()
               .HasDatabaseName("IX_Users_Email")
               .HasFilter("[IsDeleted] = 0"); // partial index

        builder.HasIndex(u => new { u.TenantId, u.CreatedAt })
               .HasDatabaseName("IX_Users_Tenant_Created");

        // Concurrency token (optimistic concurrency)
        builder.Property(u => u.RowVersion)
               .IsRowVersion();

        // Default values
        builder.Property(u => u.CreatedAt)
               .HasDefaultValueSql("SYSUTCDATETIME()");

        builder.Property(u => u.IsActive)
               .HasDefaultValue(true);

        // Owned type (value object — stored in same table)
        builder.OwnsOne(u => u.Address, address =>
        {
            address.Property(a => a.Street).HasMaxLength(200).HasColumnName("address_street");
            address.Property(a => a.City).HasMaxLength(100).HasColumnName("address_city");
            address.Property(a => a.Country).HasMaxLength(100).HasColumnName("address_country");
            address.Property(a => a.PostalCode).HasMaxLength(20).HasColumnName("address_postal_code");
        });

        // Relationship
        builder.HasMany(u => u.Orders)
               .WithOne(o => o.User)
               .HasForeignKey(o => o.UserId)
               .OnDelete(DeleteBehavior.Restrict); // NEVER cascade in enterprise
    }
}
```

#### Auto-apply all configurations
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Discovers and applies all IEntityTypeConfiguration<T> in the assembly
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

    // Global soft-delete filter
    modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
    modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted);
}
```

---

### Conventions — Customizing Defaults (EF Core 10)

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configBuilder)
{
    // All strings: max 256 chars (prevents nvarchar(max) everywhere)
    configBuilder.Properties<string>().HaveMaxLength(256);

    // All decimals: consistent precision
    configBuilder.Properties<decimal>().HavePrecision(18, 4);

    // All DateTimeOffset: store as UTC
    configBuilder.Properties<DateTimeOffset>()
                 .HaveConversion<DateTimeOffsetToBinaryConverter>();

    // Remove FK index convention (manage indexes explicitly)
    configBuilder.Conventions.Remove(typeof(ForeignKeyIndexConvention));
}
```

---

### Relationship Mapping

#### One-to-Many
```csharp
// User has many Orders
builder.HasMany(u => u.Orders)
       .WithOne(o => o.User)
       .HasForeignKey(o => o.UserId)
       .IsRequired()
       .OnDelete(DeleteBehavior.Restrict);
```

#### Many-to-Many (EF Core auto-creates join table)
```csharp
// Product has many Tags, Tag has many Products
builder.HasMany(p => p.Tags)
       .WithMany(t => t.Products)
       .UsingEntity<ProductTag>(
           pt => pt.HasOne(x => x.Tag).WithMany().HasForeignKey(x => x.TagId),
           pt => pt.HasOne(x => x.Product).WithMany().HasForeignKey(x => x.ProductId),
           pt =>
           {
               pt.ToTable("ProductTags");
               pt.HasKey(x => new { x.ProductId, x.TagId });
           });
```

#### One-to-One
```csharp
builder.HasOne(u => u.Profile)
       .WithOne(p => p.User)
       .HasForeignKey<UserProfile>(p => p.UserId)
       .OnDelete(DeleteBehavior.Cascade); // profile is owned by user
```

#### Table-per-Hierarchy (TPH) — polymorphism in one table
```csharp
builder.HasDiscriminator<string>("PaymentType")
       .HasValue<CreditCardPayment>("CreditCard")
       .HasValue<BankTransferPayment>("BankTransfer")
       .HasValue<CryptoPayment>("Crypto");
```

#### Table-per-Type (TPT) — each type in its own table
```csharp
builder.UseTptMappingStrategy();
```

---

### Querying — The Full EF Core LINQ Toolkit

```csharp
// Tracking (default) — use for writes
var user = await context.Users.FirstOrDefaultAsync(u => u.Id == id, ct);

// No-tracking (read-only) — faster, no identity map overhead
var user = await context.Users.AsNoTracking()
               .FirstOrDefaultAsync(u => u.Id == id, ct);

// AsNoTrackingWithIdentityResolution — no tracking but deduplicates navigations
var users = await context.Users.AsNoTrackingWithIdentityResolution()
               .Include(u => u.Orders)
               .ToListAsync(ct);

// Eager loading — JOIN in SQL
var user = await context.Users
    .Include(u => u.Orders)
        .ThenInclude(o => o.Items)
    .FirstOrDefaultAsync(u => u.Id == id, ct);

// Split queries — avoids cartesian explosion for multiple Includes
var user = await context.Users
    .AsSplitQuery()
    .Include(u => u.Orders)
    .Include(u => u.Addresses)
    .FirstOrDefaultAsync(u => u.Id == id, ct);

// Projection — only select what you need (most performant)
var dto = await context.Users
    .Where(u => u.Id == id)
    .Select(u => new UserDto(u.Id, u.Name, u.Email))
    .FirstOrDefaultAsync(ct);

// Paging
var page = await context.Users
    .AsNoTracking()
    .OrderBy(u => u.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(ct);

// Count + data in one trip (two queries)
var totalCount = await context.Users.CountAsync(filter, ct);
var items = await context.Users.Where(filter)
    .Skip(skip).Take(take).ToListAsync(ct);

// Compiled query (cache LINQ translation — best for hot read paths)
private static readonly Func<AppDbContext, Guid, Task<User?>> GetUserByIdQuery =
    EF.CompileAsyncQuery((AppDbContext ctx, Guid id) =>
        ctx.Users.FirstOrDefault(u => u.Id == id));

// Aggregate functions
var stats = await context.Orders
    .GroupBy(o => o.UserId)
    .Select(g => new
    {
        UserId = g.Key,
        Count = g.Count(),
        Total = g.Sum(o => o.Amount),
        Average = g.Average(o => o.Amount)
    })
    .ToListAsync(ct);

// Raw SQL (fully parameterized — no injection risk)
var users = await context.Users
    .FromSqlInterpolated($"SELECT * FROM identity.Users WHERE Email LIKE {pattern}")
    .AsNoTracking()
    .ToListAsync(ct);

// Query tags (appear in SQL profiler / App Insights)
var result = await context.Users
    .TagWith("GetActiveUsers-UserService")
    .Where(u => u.IsActive)
    .ToListAsync(ct);
```

---

### Saving Data

```csharp
// Add single
await context.Users.AddAsync(user, ct);
await context.SaveChangesAsync(ct);

// Add many (batched insert)
await context.Users.AddRangeAsync(users, ct);
await context.SaveChangesAsync(ct);

// Update (tracked entity)
user.UpdateProfile(newName, newEmail);   // domain method
context.Users.Update(user);
await context.SaveChangesAsync(ct);

// Soft delete
user.SoftDelete();
await context.SaveChangesAsync(ct);     // EF sees IsDeleted change automatically

// Bulk update — single SQL UPDATE, no entity loading
await context.Users
    .Where(u => u.LastLoginAt < thirtyDaysAgo)
    .ExecuteUpdateAsync(s =>
        s.SetProperty(u => u.IsActive, false)
         .SetProperty(u => u.UpdatedAt, DateTimeOffset.UtcNow), ct);

// Bulk delete — single SQL DELETE, no entity loading
await context.Orders
    .Where(o => o.Status == OrderStatus.Cancelled && o.CreatedAt < cutoff)
    .ExecuteDeleteAsync(ct);

// Upsert (EF Core 10)
await context.Users.Upsert(user)
    .On(u => u.Email)  // conflict key
    .WhenMatched(u => u.SetProperty(x => x.UpdatedAt, DateTimeOffset.UtcNow))
    .RunAsync(ct);
```

---

### Change Tracking — Understanding the State Machine

```
Detached → Added → Unchanged → Modified → Deleted
                   (SaveChanges inserts/updates/deletes)
```

```csharp
// Check state
var entry = context.Entry(user);
Console.WriteLine(entry.State); // Unchanged, Modified, Added, Deleted, Detached

// Manually set state (for disconnected scenarios)
context.Attach(user);                        // Unchanged — no SQL
context.Entry(user).State = EntityState.Modified;  // will UPDATE on SaveChanges

// Access original vs current values
var original = entry.OriginalValues["Email"];
var current = entry.CurrentValues["Email"];

// Detect what changed
foreach (var property in entry.Properties.Where(p => p.IsModified))
{
    Console.WriteLine($"{property.Metadata.Name}: {property.OriginalValue} → {property.CurrentValue}");
}
```

---

### Migrations — The Full Workflow

```bash
# Install EF tools (once globally)
dotnet tool install --global dotnet-ef

# Add migration (always give descriptive names!)
dotnet ef migrations add AddUserPhoneNumber \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api \
  --context AppDbContext

# List migrations and their status
dotnet ef migrations list \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Apply to local dev database
dotnet ef database update \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Generate idempotent SQL script (for production deployment review)
dotnet ef migrations script \
  --idempotent \
  --output ./migrations/$(date +%Y%m%d_%H%M%S)_migration.sql \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api

# Roll back to specific migration
dotnet ef database update PreviousMigrationName \
  --project src/MyApi.Infrastructure

# Remove last migration (if not applied to DB)
dotnet ef migrations remove \
  --project src/MyApi.Infrastructure \
  --startup-project src/MyApi.Api
```

#### Apply Migrations at Runtime (safe approach for containerized apps)
```csharp
// Only use this approach if you know what you're doing
// Prefer pipeline-driven SQL scripts in enterprise
public static async Task ApplyMigrationsAsync(IServiceProvider services)
{
    using var scope = services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    // Check for pending migrations before applying
    var pending = await context.Database.GetPendingMigrationsAsync();
    if (pending.Any())
    {
        await context.Database.MigrateAsync();
    }
}
```

---

### EF Core Performance Best Practices

| Problem | Solution |
|---|---|
| N+1 queries | Use `.Include()` or `.AsSplitQuery()` |
| Tracking read-only data | `.AsNoTracking()` always on reads |
| Selecting entire entity when projection suffices | `.Select(x => new Dto(...))` |
| String columns defaulting to `nvarchar(max)` | Set `HaveMaxLength()` in conventions |
| Decimal defaulting to `decimal(18,2)` | Set `HavePrecision(18,4)` in conventions |
| Hot path LINQ retranslation | `EF.CompileAsyncQuery()` |
| Bulk ops loading entities | `ExecuteUpdateAsync` / `ExecuteDeleteAsync` |
| Cartesian explosion with multiple Includes | `.AsSplitQuery()` |
| Connection pool exhaustion | Scope `DbContext` per request, use DI correctly |

---

### EF Core — Complete DbContext Setup

```csharp
// AppDbContext.cs
public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    // DbSets — the gateway to each table
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

        // Global soft-delete filters
        modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
        modelBuilder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted);
        modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
    }

    protected override void ConfigureConventions(ModelConfigurationBuilder configBuilder)
    {
        configBuilder.Properties<string>().HaveMaxLength(256);
        configBuilder.Properties<decimal>().HavePrecision(18, 4);
    }
}
```

---

## PART 2: EF CORE + MONGODB PROVIDER (ODM)

### Package
```xml
<PackageReference Include="MongoDB.EntityFrameworkCore" Version="8.*" />
<!-- also needs: -->
<PackageReference Include="MongoDB.Driver" Version="3.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.*" />
```

### When to Use EF Core MongoDB Provider vs Native Driver

| Use EF Core MongoDB Provider | Use Native MongoDB Driver |
|---|---|
| Team already knows EF Core patterns | Need full aggregation pipeline |
| Want LINQ over MongoDB | Need Atlas Search (`$search`) |
| Simple CRUD on collections | Need Atlas Vector Search |
| Want `SaveChanges()` change tracking | Need `$facet`, `$bucket`, `$graphLookup` |
| Code-first schema design | High-throughput writes with bulk operations |
| Unified API across SQL + Mongo | Need granular write concerns per operation |

### DbContext Setup for MongoDB
```csharp
public sealed class CatalogDbContext : DbContext
{
    public DbSet<Product> Products { get; init; } = null!;
    public DbSet<Category> Categories { get; init; } = null!;

    // Static factory (preferred pattern for MongoDB EF provider)
    public static CatalogDbContext Create(IMongoDatabase database) =>
        new(new DbContextOptionsBuilder<CatalogDbContext>()
            .UseMongoDB(database.Client, database.DatabaseNamespace.DatabaseName)
            .Options);

    public CatalogDbContext(DbContextOptions<CatalogDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Map entity to MongoDB collection name
        modelBuilder.Entity<Product>().ToCollection("products");
        modelBuilder.Entity<Category>().ToCollection("categories");
    }
}
```

### Register in DI (Program.cs)
```csharp
builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var connectionString = builder.Configuration["MongoDb:ConnectionString"];
    return new MongoClient(connectionString);
});

builder.Services.AddDbContext<CatalogDbContext>((sp, options) =>
{
    var client = sp.GetRequiredService<IMongoClient>();
    var dbName = builder.Configuration["MongoDb:DatabaseName"];
    options.UseMongoDB(client, dbName!);
});
```

### Entity Models for MongoDB EF Provider
```csharp
public class Product
{
    // ObjectId as primary key
    [BsonId]
    public ObjectId Id { get; set; } = ObjectId.GenerateNewId();

    [BsonElement("name")]
    public string Name { get; set; } = default!;

    [BsonElement("price")]
    [BsonRepresentation(BsonType.Decimal128)]
    public decimal Price { get; set; }

    [BsonElement("tags")]
    public List<string> Tags { get; set; } = [];

    // Embedded document — stored as nested object in same document
    [BsonElement("specifications")]
    public ProductSpecifications Specifications { get; set; } = new();

    [BsonElement("createdAt")]
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    [BsonIgnoreIfNull]
    public DateTime? UpdatedAt { get; set; }
}

// Embedded/owned type — no ToCollection(), lives inside Product document
public class ProductSpecifications
{
    [BsonElement("weight")]
    public double WeightKg { get; set; }

    [BsonElement("color")]
    public string Color { get; set; } = default!;
}
```

### CRUD Operations — MongoDB EF Provider Syntax
```csharp
// ─── CREATE ───
// Single insert
var product = new Product { Name = "Widget", Price = 9.99m };
db.Products.Add(product);
await db.SaveChangesAsync(ct);

// Bulk insert
db.Products.AddRange(products);
await db.SaveChangesAsync(ct);

// ─── READ ───
// Find one
var product = await db.Products
    .FirstOrDefaultAsync(p => p.Name == "Widget", ct);

// Find many with filter
var cheap = await db.Products
    .Where(p => p.Price < 50m && p.Tags.Contains("sale"))
    .OrderBy(p => p.Price)
    .ToListAsync(ct);

// Skip/Take (pagination)
var page = await db.Products
    .OrderBy(p => p.CreatedAt)
    .Skip((pageNum - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(ct);

// Projection (select specific fields)
var names = await db.Products
    .Select(p => new { p.Id, p.Name, p.Price })
    .ToListAsync(ct);

// ─── UPDATE ───
// Load, modify, save (change tracking handles the rest)
var product = await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
if (product is not null)
{
    product.Price = newPrice;
    product.UpdatedAt = DateTime.UtcNow;
    await db.SaveChangesAsync(ct);
}

// Bulk update multiple entities
var products = db.Products.Where(p => p.Tags.Contains("clearance"));
foreach (var p in products) { p.Price *= 0.8m; }
await db.SaveChangesAsync(ct);

// ─── DELETE ───
var product = await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
if (product is not null)
{
    db.Products.Remove(product);
    await db.SaveChangesAsync(ct);
}

// Bulk delete
var discontinued = db.Products.Where(p => p.IsDiscontinued);
db.Products.RemoveRange(discontinued);
await db.SaveChangesAsync(ct);

// ─── SORT & ORDER ───
var sorted = db.Products
    .OrderBy(p => p.Price)
    .ThenByDescending(p => p.CreatedAt)
    .ToList();
```

---

## PART 3: ODM ECOSYSTEM — OTHER LANGUAGES (FYI)

### For .NET / C#
| ODM/ORM | Status | Best For |
|---|---|---|
| **EF Core + SQL Server provider** | Official Microsoft | SQL Server, relational |
| **EF Core + MongoDB provider** | Official MongoDB | MongoDB, teams who prefer EF |
| **Native MongoDB C# Driver** | Official MongoDB | Full MongoDB feature access |

### Python ODMs
| Library | Type | Notes |
|---|---|---|
| **Beanie** | Async ODM | Pydantic-based, best for FastAPI |
| **MongoEngine** | Sync ORM | Declarative API, built on PyMongo |
| **Django MongoDB Backend** | ORM integration | Official; works with Django ORM |

### Node.js ODMs
| Library | Notes |
|---|---|
| **Mongoose** | Schema enforcement at app layer, hooks, validation — most popular |
| **Prisma** | Declarative schema, type-safe, modern; supports MongoDB + relational |

### Java / Spring
| Library | Notes |
|---|---|
| **Spring Data MongoDB** | POJO-centric, repository pattern, LINQ-like |
| **Hibernate ORM + MongoDB Extension** | Official MongoDB extension for Hibernate |

### Ruby
| Library | Notes |
|---|---|
| **Mongoid** | ActiveRecord-compatible API for MongoDB |

### PHP
| Library | Notes |
|---|---|
| **Doctrine MongoDB ODM** | GridFS support, embedded/referenced documents |
| **Laravel MongoDB** | Extends Eloquent for MongoDB |

---

## PART 4: EF CORE PROVIDER COMPARISON TABLE

| Feature | EF Core + SQL Server | EF Core + MongoDB |
|---|---|---|
| LINQ queries | Full support | Supported subset |
| Migrations | Full (schema evolution) | ❌ Not supported (schemaless) |
| Change tracking | ✅ Full | ✅ Full |
| Transactions | ✅ ACID, multi-table | ✅ Multi-document ACID |
| Relationships/Joins | ✅ Native FK + JOIN | ⚠️ Embedded docs or app-level joins |
| Raw SQL / Aggregation | `FromSqlRaw()` | ⚠️ Use native driver for `$search`, `$facet` |
| Schema validation | DB-level constraints | App-level only |
| Inheritance mapping | TPH, TPT, TPC | Limited |
| Concurrency tokens | RowVersion / timestamp | ❌ Not natively supported |
| Bulk ExecuteUpdate/Delete | ✅ EF Core 10 | ❌ Use native driver |
| Index management | Via migrations | Via native driver `Indexes.CreateManyAsync` |

---

## PART 5: ENTERPRISE DECISION — WHICH API TO USE

```
Building a relational data model?
  → EF Core + SQL Server provider
  → Full migrations, relationships, ACID, reporting

Building document storage where team knows EF Core?
  → EF Core + MongoDB provider
  → Familiar LINQ API, change tracking, simpler CRUD

Building document storage needing full MongoDB power?
  → Native MongoDB C# Driver (MongoDB.Driver)
  → Aggregation pipelines, Atlas Search, Vector Search, bulk ops

Building the same app needing BOTH?
  → EF Core (AppDbContext) for SQL Server
  → Native Driver (IMongoCollection<T>) for MongoDB
  → Each registered separately in DI — clear separation
```

---

## PART 6: UNIFIED SERVICE LAYER PATTERN

```csharp
// When your app uses BOTH databases

// SQL Server service (EF Core)
public class UserService(AppDbContext context, IUnitOfWork uow)
{
    public async Task<UserDto?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await context.Users
            .AsNoTracking()
            .Where(u => u.Id == id)
            .Select(u => new UserDto(u.Id, u.Name, u.Email))
            .FirstOrDefaultAsync(ct);
    }
}

// MongoDB service (native driver or EF Core provider)
public class ProductService(CatalogDbContext catalog)
{
    public async Task<List<Product>> SearchByTagAsync(string tag, CancellationToken ct)
    {
        return await catalog.Products
            .Where(p => p.Tags.Contains(tag))
            .OrderBy(p => p.Name)
            .ToListAsync(ct);
    }
}

// DI Registration (Program.cs)
builder.Services.AddDbContext<AppDbContext>(...);      // SQL Server
builder.Services.AddDbContext<CatalogDbContext>(...);  // MongoDB
builder.Services.AddScoped<UserService>();
builder.Services.AddScoped<ProductService>();
```

---

_Sources: learn.microsoft.com/ef/core, mongodb.com/docs/entity-framework, mongodb.com/docs/drivers/odm_
