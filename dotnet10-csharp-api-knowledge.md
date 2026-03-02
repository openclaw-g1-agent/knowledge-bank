# .NET 10 / C# 14 — Enterprise API Development Knowledge Base
_Compiled: 2026-02-28_

---

## 1. .NET 10 Overview

- **LTS Release** — 3-year support. Successor to .NET 9.
- Ships with **C# 14**, **ASP.NET Core 10**, **EF Core 10**, **Aspire 13.1**
- Download: https://get.dot.net/10

---

## 2. Runtime Improvements

- Improved **JIT inlining**, **method devirtualization**, **stack allocations**
- **AVX10.2** vector support
- **NativeAOT** enhancements (smaller binaries, faster startup)
- Better code generation for struct arguments
- Enhanced loop inversion optimization

---

## 3. C# 14 — New Language Features

### Extension Members (Big One)
```csharp
public static class Enumerable
{
    extension<TSource>(IEnumerable<TSource> source)  // instance extensions
    {
        public bool IsEmpty => !source.Any();
        public IEnumerable<TSource> Where(Func<TSource, bool> predicate) { ... }
    }

    extension<TSource>(IEnumerable<TSource>)  // static extensions
    {
        public static IEnumerable<TSource> Identity => Enumerable.Empty<TSource>();
        public static IEnumerable<TSource> operator +(IEnumerable<TSource> left, IEnumerable<TSource> right) => left.Concat(right);
    }
}
```

### `field` Keyword (Field-Backed Properties)
```csharp
// Before
private string _msg;
public string Message { get => _msg; set => _msg = value ?? throw new ArgumentNullException(nameof(value)); }

// After (C# 14)
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

### Null-Conditional Assignment
```csharp
// Before
if (customer is not null) customer.Order = GetCurrentOrder();

// After
customer?.Order = GetCurrentOrder();
customer?.Order ??= GetCurrentOrder(); // compound assignment too
```

### Span<T> First-Class Support
- Implicit conversions between `Span<T>`, `ReadOnlySpan<T>`, `T[]`
- Works as extension method receivers
- Better generic type inference with spans

### Lambda Parameter Modifiers (no type required)
```csharp
TryParse<int> parse = (text, out result) => Int32.TryParse(text, out result);
// No need to write (string text, out int result)
```

### `nameof` with Unbound Generics
```csharp
nameof(List<>)  // returns "List"
nameof(Dictionary<,>)  // returns "Dictionary"
```

### Partial Constructors and Events
```csharp
// Defining declaration
partial class MyService
{
    public partial MyService(ILogger logger);  // defining
}

// Implementing declaration (different file)
partial class MyService
{
    public partial MyService(ILogger logger)  // implementing
    {
        _logger = logger;
        InitializeDefaults();
    }
}
```

### User-Defined Compound Assignment Operators
```csharp
public static MyVector operator +(MyVector left, MyVector right) => ...;
// Now: myVec += other;  works automatically via user-defined compound assignment
```

---

## 4. ASP.NET Core 10 — API Development

### Recommended Approach: Minimal APIs
Microsoft **recommends Minimal APIs for new projects** as of .NET 10.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/users/{id}", async (int id, IUserService svc) => 
    await svc.GetByIdAsync(id) is { } user 
        ? Results.Ok(user) 
        : Results.NotFound());

app.Run();
```

### Controller-Based APIs (still supported)
Use when you need:
- Model binding extensibility (`IModelBinderProvider`, `IModelBinder`)
- `JsonPatch` support
- OData support
- Application parts or application model features

### Key ASP.NET Core 10 Features
- **OpenAPI enhancements** — better schema generation
- **Passkey/WebAuthn support** in ASP.NET Identity
- **Blazor script as static web asset** — automatic compression + fingerprinting
- **Client-side fingerprinting** for Blazor WASM
- **HttpClient streaming enabled by default** in Blazor
- New `ReconnectModal` component for SignalR reconnection UI (CSP-compliant)
- `QuickGrid`: `RowClass` parameter, `HideColumnOptionsAsync()` method

---

## 5. .NET 10 Libraries — Key APIs

### Cryptography
- **Post-Quantum Cryptography (PQC)**: ML-KEM (FIPS 203), ML-DSA (FIPS 204), SLH-DSA (FIPS 205)
  ```csharp
  using MLKem key = MLKem.GenerateKey(MLKemAlgorithm.MLKem768);
  using MLDsa key = MLDsa.ImportFromPem(publicKeyPem);
  ```
- **Certificate thumbprint search by algorithm** (not just SHA-1):
  ```csharp
  var coll = store.Certificates.FindByThumbprint(HashAlgorithmName.SHA256, thumbprint);
  ```
- **PEM encoding for UTF-8/ASCII bytes** (skip char conversion):
  ```csharp
  PemFields fields = PemEncoding.FindUtf8(fileBytes);
  ```
- **PKCS#12 export** with modern AES-256 + SHA-256 (not just legacy 3DES):
  ```csharp
  cert.ExportPkcs12(Pkcs12ExportPbeParameters.Pbes2Aes256Sha256, password);
  ```

### JSON / Serialization
- **Disallow duplicate JSON properties** option
- **Strict serialization settings**
- **PipeReader support** for JSON deserialization (streaming, zero-copy)

### Networking
- **`WebSocketStream`** — wraps WebSocket as a standard Stream
- **TLS 1.3** client support on macOS

### Collections & Diagnostics
- New APIs in numerics, globalization, ZIP files

### Process Management
- Windows process group support for better signal/job isolation

---

## 6. .NET 10 SDK

- `dotnet test` integrates **Microsoft.Testing.Platform**
- Standardized CLI command ordering
- **Native tab-completion** scripts for bash/zsh/fish/PowerShell
- Console apps can natively **create container images**
- `dotnet tool exec` — one-shot tool execution without install
- `dnx` script for tool execution
- `--cli-schema` for CLI introspection
- **Platform-specific .NET tools** with `any` RuntimeIdentifier
- **File-based app publish** + NativeAOT support

---

## 7. EF Core 10

- Named **query filters** (multiple per entity, selective disable)
- LINQ enhancements + performance optimizations
- Improved Azure Cosmos DB support

---

## 8. Enterprise API Development Best Practices (.NET 10)

### Project Structure
```
MyApi/
├── src/
│   ├── MyApi.Api/                  # Entry point, DI setup, middleware
│   │   ├── Program.cs
│   │   ├── Endpoints/              # Minimal API endpoint groups
│   │   └── Middleware/
│   ├── MyApi.Application/          # Business logic, CQRS handlers
│   │   ├── Commands/
│   │   ├── Queries/
│   │   └── Interfaces/
│   ├── MyApi.Domain/               # Domain models, value objects
│   └── MyApi.Infrastructure/       # Data access, external services
│       ├── Persistence/
│       └── ExternalServices/
└── tests/
    ├── MyApi.UnitTests/
    ├── MyApi.IntegrationTests/
    └── MyApi.ArchitectureTests/
```

### Program.cs Pattern (Enterprise Minimal API)
```csharp
var builder = WebApplication.CreateBuilder(args);

// Service registration — use extension methods per feature
builder.Services
    .AddApplication()          // MediatR, FluentValidation, AutoMapper
    .AddInfrastructure(builder.Configuration)  // DbContext, Repos, External
    .AddApiServices();         // OpenAPI, Auth, CORS, Rate Limiting

// Add OpenAPI (replaces Swashbuckle in .NET 10+)
builder.Services.AddOpenApi();

var app = builder.Build();

// Middleware pipeline (order matters!)
if (app.Environment.IsDevelopment())
    app.MapOpenApi();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

// Endpoint groups
app.MapGroup("/api/v1")
   .MapUsersEndpoints()
   .MapOrdersEndpoints()
   .RequireAuthorization();

app.Run();
```

### Endpoint Group Pattern
```csharp
public static class UsersEndpoints
{
    public static RouteGroupBuilder MapUsersEndpoints(this RouteGroupBuilder group)
    {
        var users = group.MapGroup("/users")
            .WithTags("Users")
            .WithOpenApi();

        users.MapGet("/", GetAllUsers);
        users.MapGet("/{id:guid}", GetUserById);
        users.MapPost("/", CreateUser).RequireAuthorization("Admin");
        users.MapPut("/{id:guid}", UpdateUser).RequireAuthorization("Admin");
        users.MapDelete("/{id:guid}", DeleteUser).RequireAuthorization("Admin");

        return group;
    }

    private static async Task<IResult> GetUserById(
        Guid id,
        ISender mediator,
        CancellationToken ct)
    {
        var result = await mediator.Send(new GetUserByIdQuery(id), ct);
        return result.Match(
            user => Results.Ok(user),
            _ => Results.NotFound());
    }

    private static async Task<IResult> CreateUser(
        CreateUserRequest request,
        ISender mediator,
        IValidator<CreateUserRequest> validator,
        CancellationToken ct)
    {
        var validation = await validator.ValidateAsync(request, ct);
        if (!validation.IsValid)
            return Results.ValidationProblem(validation.ToDictionary());

        var result = await mediator.Send(new CreateUserCommand(request), ct);
        return result.Match(
            user => Results.CreatedAtRoute("GetUserById", new { id = user.Id }, user),
            errors => Results.Problem(errors.ToProblemDetails()));
    }
}
```

### API Versioning
```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.ReportApiVersions = true;
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
});
```

### Global Exception Handling (ProblemDetails)
```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

// GlobalExceptionHandler.cs
public sealed class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        var (statusCode, title) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Not Found"),
            ValidationException => (StatusCodes.Status400BadRequest, "Validation Error"),
            UnauthorizedException => (StatusCodes.Status401Unauthorized, "Unauthorized"),
            ForbiddenException => (StatusCodes.Status403Forbidden, "Forbidden"),
            ConflictException => (StatusCodes.Status409Conflict, "Conflict"),
            _ => (StatusCodes.Status500InternalServerError, "Internal Server Error")
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = title,
            Status = statusCode,
            Detail = exception.Message,
            Instance = context.Request.Path
        }, ct);

        return true;
    }
}
```

### Authentication & Authorization
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.Zero
        };
    });

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("Admin", policy => policy.RequireRole("admin"))
    .AddPolicy("ReadOnly", policy => policy.RequireClaim("scope", "read"));
```

### Rate Limiting
```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", limiter =>
    {
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.PermitLimit = 100;
        limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiter.QueueLimit = 10;
    });

    options.AddSlidingWindowLimiter("sliding", limiter =>
    {
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.SegmentsPerWindow = 6;
        limiter.PermitLimit = 100;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});
```

### Logging & Observability (OpenTelemetry)
```csharp
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
});

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddOtlpExporter());
```

### Health Checks
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database")
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!, "redis")
    .AddUrlGroup(new Uri("https://externalservice.com/health"), "external");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false
});
```

### CQRS Pattern with MediatR
```csharp
// Command
public record CreateUserCommand(CreateUserRequest Request) : IRequest<Result<UserDto>>;

// Handler
public class CreateUserCommandHandler : IRequestHandler<CreateUserCommand, Result<UserDto>>
{
    private readonly IUserRepository _repo;
    private readonly IUnitOfWork _uow;

    public CreateUserCommandHandler(IUserRepository repo, IUnitOfWork uow)
    {
        _repo = repo;
        _uow = uow;
    }

    public async Task<Result<UserDto>> Handle(CreateUserCommand cmd, CancellationToken ct)
    {
        if (await _repo.ExistsByEmailAsync(cmd.Request.Email, ct))
            return Result.Failure<UserDto>(UserErrors.EmailAlreadyExists);

        var user = User.Create(cmd.Request.Name, cmd.Request.Email);
        await _repo.AddAsync(user, ct);
        await _uow.SaveChangesAsync(ct);

        return Result.Success(user.ToDto());
    }
}

// Validation pipeline behavior
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next();

        var errors = validators
            .Select(v => v.Validate(request))
            .SelectMany(r => r.Errors)
            .Where(e => e is not null)
            .ToList();

        if (errors.Count > 0)
            throw new ValidationException(errors);

        return await next();
    }
}
```

### Repository Pattern with EF Core 10
```csharp
public class UserRepository(AppDbContext context) : IUserRepository
{
    public async Task<User?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        await context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, ct);

    public async Task<bool> ExistsByEmailAsync(string email, CancellationToken ct = default) =>
        await context.Users.AnyAsync(u => u.Email == email, ct);

    public async Task AddAsync(User user, CancellationToken ct = default) =>
        await context.Users.AddAsync(user, ct);
}
```

### OpenAPI / Swagger (.NET 10 built-in)
```csharp
// .NET 10 ships with Microsoft.AspNetCore.OpenApi (replaces Swashbuckle)
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((doc, ctx, ct) =>
    {
        doc.Info.Title = "My Enterprise API";
        doc.Info.Version = "v1";
        doc.Info.Contact = new() { Name = "Team", Email = "api@company.com" };
        return Task.CompletedTask;
    });

    options.AddOperationTransformer<BearerSecuritySchemeTransformer>();
});

// Endpoint decoration
users.MapGet("/{id:guid}", GetUserById)
    .WithName("GetUserById")
    .WithSummary("Get user by ID")
    .WithDescription("Returns a single user by their unique identifier.")
    .Produces<UserDto>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound)
    .WithOpenApi();
```

### Response Caching & Output Caching
```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Cache());
    options.AddPolicy("UserCache", builder =>
        builder.Tag("users")
               .Expire(TimeSpan.FromMinutes(5))
               .VaryByHeader("Accept-Language"));
});

// On endpoint:
users.MapGet("/", GetAllUsers)
    .CacheOutput("UserCache");

// Invalidate cache when data changes:
await cache.EvictByTagAsync("users", ct);
```

### Background Services
```csharp
public class EmailDispatchWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<EmailDispatchWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var emailSvc = scope.ServiceProvider.GetRequiredService<IEmailService>();

            await emailSvc.ProcessPendingEmailsAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}
```

---

## 9. Enterprise Code Quality Standards

### Naming Conventions
- **Interfaces**: `IUserService`, `IRepository<T>`
- **Records**: `CreateUserRequest`, `UserDto`
- **Commands/Queries**: `CreateUserCommand`, `GetUserByIdQuery`
- **Handlers**: `CreateUserCommandHandler`
- **Async methods**: always suffix `Async`
- **Private fields**: `_camelCase`
- **Constants**: `UPPER_CASE` or `PascalCase` (prefer PascalCase in C#)

### Use Records for DTOs and Commands
```csharp
public record UserDto(Guid Id, string Name, string Email, DateTimeOffset CreatedAt);
public record CreateUserRequest(string Name, string Email, string Password);
public record UpdateUserRequest(string? Name, string? Email);
```

### Use Result<T> Pattern (not exceptions for flow control)
```csharp
public sealed class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error? Error { get; }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(Error error) => new(false, default, error);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}
```

### Nullable Reference Types — Always Enabled
```xml
<Nullable>enable</Nullable>
<ImplicitUsings>enable</ImplicitUsings>
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
```

### Cancellation Token — Always Thread Through
```csharp
public async Task<UserDto?> GetByIdAsync(Guid id, CancellationToken ct = default)
{
    return await _repo.GetByIdAsync(id, ct);
}
```

### FluentValidation
```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required.")
            .MaximumLength(100).WithMessage("Name must not exceed 100 characters.");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress().WithMessage("A valid email is required.");

        RuleFor(x => x.Password)
            .MinimumLength(8).WithMessage("Password must be at least 8 characters.")
            .Matches("[A-Z]").WithMessage("Must contain uppercase letter.")
            .Matches("[0-9]").WithMessage("Must contain a digit.");
    }
}
```

---

## 10. Security Best Practices

- Always use **HTTPS** (`UseHttpsRedirection`)
- Use **JWT with short expiry** (15 min) + **refresh tokens**
- Store secrets in **Azure Key Vault** / environment variables (never in code)
- Use **`IOptions<T>` pattern** for strongly-typed config
- Enable **CORS** explicitly — never wildcard in production
- Apply **rate limiting** to prevent abuse
- Use **parameterized queries** via EF Core (no raw SQL interpolation)
- Validate **all inputs** — never trust client data
- Enable **output caching** carefully — never cache auth-sensitive data
- Use **`[Authorize]`** by default; whitelist public endpoints
- Implement **RBAC** (role-based) + **claims-based** authorization policies
- Use **post-quantum crypto** for long-lived secrets (ML-KEM, ML-DSA) where needed

---

## 11. Testing Enterprise APIs

```csharp
// Integration test with WebApplicationFactory
public class UsersEndpointsTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UsersEndpointsTests(WebApplicationFactory<Program> factory)
    {
        _client = factory
            .WithWebHostBuilder(builder =>
                builder.ConfigureServices(services =>
                {
                    // Replace real DbContext with in-memory
                    services.RemoveAll<AppDbContext>();
                    services.AddDbContext<AppDbContext>(opt =>
                        opt.UseInMemoryDatabase("TestDb"));
                }))
            .CreateClient();
    }

    [Fact]
    public async Task GetUser_ReturnsNotFound_WhenUserDoesNotExist()
    {
        var response = await _client.GetAsync($"/api/v1/users/{Guid.NewGuid()}");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

---

_Sources: learn.microsoft.com/dotnet/core/whats-new/dotnet-10, aspnetcore-10.0 release notes, C# 14 release notes_
