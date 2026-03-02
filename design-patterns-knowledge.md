# Gang of Four Design Patterns — Complete Knowledge Base
_Compiled: 2026-02-28 | Source: refactoring.guru + GoF book_

---

## Overview

The **Gang of Four (GoF)** refers to the four authors of _"Design Patterns: Elements of Reusable Object-Oriented Software"_ (1994): Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides.

**23 patterns** organized into **3 categories**:

| Category | Purpose | Count |
|---|---|---|
| **Creational** | How objects are created | 5 |
| **Structural** | How objects/classes are composed into larger structures | 7 |
| **Behavioral** | How objects communicate and assign responsibility | 11 |

**Core principle:** Program to an **interface**, not an implementation. Favor **composition** over inheritance.

---

## CREATIONAL PATTERNS

> _"These patterns provide various object creation mechanisms, which increase flexibility and reuse of existing code."_

---

### 1. Factory Method
**Also known as:** Virtual Constructor

**Intent:** Define an interface for creating an object, but let subclasses decide which class to instantiate.

**Problem:** Your code is tightly coupled to `new ConcreteClass()`. When you need different implementations, you end up with if/else chains or massive constructors.

**Solution:** Extract object creation into a factory method that subclasses override.

**Structure:**
```
Creator (abstract)            Product (interface)
├── factoryMethod(): Product  ├── doStuff()
└── someOperation()           ├── ConcreteProductA
                              └── ConcreteProductB
ConcreteCreatorA → creates ConcreteProductA
ConcreteCreatorB → creates ConcreteProductB
```

**C# Example:**
```csharp
// Product interface
public interface INotificationSender
{
    Task SendAsync(string message, string recipient, CancellationToken ct = default);
}

// Concrete products
public class EmailSender(ISmtpClient smtp) : INotificationSender
{
    public async Task SendAsync(string message, string recipient, CancellationToken ct) =>
        await smtp.SendAsync(new Email(recipient, message), ct);
}

public class SmsSender(ISmsGateway gateway) : INotificationSender
{
    public async Task SendAsync(string message, string recipient, CancellationToken ct) =>
        await gateway.SendSmsAsync(recipient, message, ct);
}

// Creator - factory method via DI in .NET is the modern approach
public interface INotificationSenderFactory
{
    INotificationSender Create(NotificationChannel channel);
}

public class NotificationSenderFactory(IServiceProvider sp) : INotificationSenderFactory
{
    public INotificationSender Create(NotificationChannel channel) => channel switch
    {
        NotificationChannel.Email => sp.GetRequiredService<EmailSender>(),
        NotificationChannel.Sms   => sp.GetRequiredService<SmsSender>(),
        NotificationChannel.Push  => sp.GetRequiredService<PushSender>(),
        _ => throw new ArgumentOutOfRangeException(nameof(channel))
    };
}
```

**When to use:**
- When you don't know ahead of time what class to instantiate
- When you want subclasses to control which objects are created
- When you want to encapsulate construction logic

**Real-world .NET examples:** `DbProviderFactory`, `HttpClientFactory`, `ILoggerFactory`

---

### 2. Abstract Factory
**Intent:** Provide an interface for creating **families** of related or dependent objects without specifying their concrete classes.

**Problem:** You need to create UI elements (buttons, checkboxes, dialogs) that must be consistent per OS (Windows vs Mac). Factory Method creates one type — Abstract Factory creates a family.

**Structure:**
```
AbstractFactory
├── CreateButton(): IButton
└── CreateCheckbox(): ICheckbox

WindowsFactory → WindowsButton + WindowsCheckbox
MacFactory     → MacButton + MacCheckbox
```

**C# Example:**
```csharp
// Abstract factory
public interface IUIFactory
{
    IButton CreateButton();
    IDialog CreateDialog();
    ITextInput CreateTextInput();
}

// Concrete factories — one per "family"
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton()    => new WindowsButton();
    public IDialog CreateDialog()    => new WindowsDialog();
    public ITextInput CreateTextInput() => new WindowsTextInput();
}

public class WebUIFactory : IUIFactory
{
    public IButton CreateButton()    => new HtmlButton();
    public IDialog CreateDialog()    => new HtmlModal();
    public ITextInput CreateTextInput() => new HtmlInput();
}

// Client code works only with interfaces — never knows which family
public class Application(IUIFactory factory)
{
    private readonly IButton _loginButton = factory.CreateButton();
    private readonly IDialog _confirmDialog = factory.CreateDialog();

    public void Render()
    {
        _loginButton.Render();
        _confirmDialog.Show();
    }
}
```

**When to use:**
- Your system needs to be independent of how its products are created
- You need to enforce that products from one family are used together
- You want to swap whole product families easily

---

### 3. Builder
**Intent:** Construct complex objects step by step, allowing different representations using the same construction code.

**Problem:** A complex object has too many optional parameters — the "telescoping constructor" anti-pattern. Or you need to create different representations of the same thing (car + car manual).

**Solution:** Extract construction into a Builder object with discrete steps. Optionally use a Director to define order.

**Structure:**
```
Director          Builder (interface)
└── construct()   ├── buildStepA()
                  ├── buildStepB()
                  └── getResult()

ConcreteBuilderA → builds ProductA
ConcreteBuilderB → builds ProductB
```

**C# Example (Fluent Builder — most common in .NET):**
```csharp
public sealed class QueryBuilder
{
    private string _table = default!;
    private readonly List<string> _conditions = [];
    private readonly List<string> _columns = ["*"];
    private int? _limit;
    private string? _orderBy;

    public QueryBuilder From(string table)  { _table = table; return this; }
    public QueryBuilder Select(params string[] cols) { _columns.Clear(); _columns.AddRange(cols); return this; }
    public QueryBuilder Where(string condition) { _conditions.Add(condition); return this; }
    public QueryBuilder Limit(int n)        { _limit = n; return this; }
    public QueryBuilder OrderBy(string col) { _orderBy = col; return this; }

    public string Build()
    {
        var sb = new StringBuilder($"SELECT {string.Join(", ", _columns)} FROM {_table}");
        if (_conditions.Any())
            sb.Append($" WHERE {string.Join(" AND ", _conditions)}");
        if (_orderBy is not null)
            sb.Append($" ORDER BY {_orderBy}");
        if (_limit.HasValue)
            sb.Append($" LIMIT {_limit}");
        return sb.ToString();
    }
}

// Usage — reads like a sentence
var query = new QueryBuilder()
    .From("users")
    .Select("id", "name", "email")
    .Where("is_active = 1")
    .Where("created_at > '2024-01-01'")
    .OrderBy("created_at DESC")
    .Limit(20)
    .Build();

// .NET real-world: WebApplicationBuilder, IHostBuilder, DbContextOptionsBuilder
```

**When to use:**
- Complex objects with many optional parameters
- When you need to create different representations of the same object
- Step-by-step construction where some steps vary

---

### 4. Prototype
**Intent:** Copy existing objects without making code dependent on their classes.

**Problem:** You need to create a copy of an object, but you don't know its concrete class and it may have private fields.

**Solution:** Delegate cloning to the object itself via a `Clone()` method.

**C# Example:**
```csharp
public abstract class Shape
{
    public int X { get; set; }
    public int Y { get; set; }
    public string Color { get; set; } = default!;

    protected Shape(Shape source) // copy constructor
    {
        X = source.X;
        Y = source.Y;
        Color = source.Color;
    }

    public abstract Shape Clone();
}

public class Circle : Shape
{
    public int Radius { get; set; }

    private Circle(Circle source) : base(source) => Radius = source.Radius;

    public override Shape Clone() => new Circle(this);
}

public class Rectangle : Shape
{
    public int Width { get; set; }
    public int Height { get; set; }

    private Rectangle(Rectangle source) : base(source)
    {
        Width = source.Width;
        Height = source.Height;
    }

    public override Shape Clone() => new Rectangle(this);
}

// Usage
var original = new Circle { X = 10, Y = 20, Radius = 50, Color = "Blue" };
var copy = original.Clone() as Circle;  // deep copy, no knowledge of concrete type
```

**When to use:**
- When object creation is expensive (DB lookup, network call) and you can clone instead
- When you need objects that only differ slightly from existing ones
- When you want to avoid rebuilding class hierarchies of factories

---

### 5. Singleton
**Intent:** Ensure a class has only one instance and provide a global access point to it.

**Problem:** Some resources (config, connection pool, logger) should have exactly one instance. Global variables are messy.

**C# Example (thread-safe, .NET idiomatic):**
```csharp
// ✅ .NET idiomatic: use DI as a singleton (preferred)
builder.Services.AddSingleton<IConfigurationService, ConfigurationService>();

// ✅ Classic implementation with Lazy<T> (thread-safe, lazy init)
public sealed class AppSettings
{
    private static readonly Lazy<AppSettings> _instance =
        new(() => new AppSettings(), LazyThreadSafetyMode.ExecutionAndPublication);

    public static AppSettings Instance => _instance.Value;

    private AppSettings() { /* load from config */ }

    public string ApiKey { get; private set; } = default!;
}

// Usage
var key = AppSettings.Instance.ApiKey;
```

**⚠️ Caveats:**
- Singletons make unit testing hard (global state)
- In .NET, **prefer DI `AddSingleton<T>()`** — same effect, but mockable and testable
- Avoid in multi-tenant scenarios

**When to use:**
- Logger instances
- Configuration/settings objects
- Connection pools
- Thread pools / scheduler

---

## STRUCTURAL PATTERNS

> _"These patterns explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient."_

---

### 6. Adapter
**Also known as:** Wrapper

**Intent:** Allow objects with incompatible interfaces to collaborate.

**Problem:** You have a class that does what you want, but its interface is incompatible with the rest of your code.

**C# Example:**
```csharp
// Existing "adaptee" — third-party library you can't change
public class LegacyPaymentGateway
{
    public string ProcessPaymentXml(string xmlPayload) => "SUCCESS"; // returns XML
}

// Target interface your system expects
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct = default);
}

// Adapter — bridges the gap
public class LegacyPaymentAdapter(LegacyPaymentGateway legacy) : IPaymentProcessor
{
    public Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct)
    {
        // Convert modern request → legacy XML
        var xml = $"<payment><amount>{request.Amount}</amount><card>{request.CardToken}</card></payment>";

        // Call legacy system
        var response = legacy.ProcessPaymentXml(xml);

        // Convert legacy XML response → modern result
        var success = response.Contains("SUCCESS");
        return Task.FromResult(new PaymentResult(success, response));
    }
}

// Client code only knows IPaymentProcessor
public class OrderService(IPaymentProcessor payments)
{
    public async Task<bool> CheckoutAsync(Order order, CancellationToken ct) =>
        (await payments.ProcessAsync(new PaymentRequest(order.Total, order.CardToken), ct)).IsSuccess;
}
```

**When to use:**
- Integrating third-party libraries with incompatible interfaces
- Wrapping legacy code behind a modern interface
- When you want to reuse existing classes that can't be modified

---

### 7. Bridge
**Intent:** Split a large class or closely related classes into two separate hierarchies — abstraction and implementation — that can vary independently.

**Problem:** You have two dimensions of variation (e.g., shape × color, platform × feature) causing a class explosion.

```csharp
// Without Bridge: CircleBlue, CircleRed, SquareBlue, SquareRed = N×M classes
// With Bridge: Shape + Color separated

public interface IRenderer // Implementation hierarchy
{
    void RenderCircle(int radius);
    void RenderRectangle(int w, int h);
}

public class SvgRenderer : IRenderer
{
    public void RenderCircle(int r) => Console.WriteLine($"<circle r='{r}'/>");
    public void RenderRectangle(int w, int h) => Console.WriteLine($"<rect width='{w}' height='{h}'/>");
}

public class CanvasRenderer : IRenderer
{
    public void RenderCircle(int r) => Console.WriteLine($"ctx.arc(0,0,{r},0,2*Math.PI)");
    public void RenderRectangle(int w, int h) => Console.WriteLine($"ctx.fillRect(0,0,{w},{h})");
}

public abstract class Shape(IRenderer renderer) // Abstraction hierarchy
{
    protected readonly IRenderer _renderer = renderer;
    public abstract void Draw();
}

public class Circle(IRenderer renderer, int radius) : Shape(renderer)
{
    public override void Draw() => _renderer.RenderCircle(radius);
}

public class Rectangle(IRenderer renderer, int w, int h) : Shape(renderer)
{
    public override void Draw() => _renderer.RenderRectangle(w, h);
}
```

---

### 8. Composite
**Intent:** Compose objects into tree structures to represent part-whole hierarchies. Lets clients treat individual objects and compositions uniformly.

**Real-world:** File system (files + folders), UI component trees, org charts.

```csharp
public interface IComponent
{
    string Name { get; }
    decimal GetSize();
    void Display(int indent = 0);
}

public class File(string name, decimal sizeKb) : IComponent
{
    public string Name => name;
    public decimal GetSize() => sizeKb;
    public void Display(int indent = 0) =>
        Console.WriteLine($"{new string(' ', indent)}📄 {Name} ({sizeKb}KB)");
}

public class Folder(string name) : IComponent
{
    private readonly List<IComponent> _children = [];
    public string Name => name;

    public void Add(IComponent component) => _children.Add(component);
    public void Remove(IComponent component) => _children.Remove(component);

    public decimal GetSize() => _children.Sum(c => c.GetSize());

    public void Display(int indent = 0)
    {
        Console.WriteLine($"{new string(' ', indent)}📁 {Name}/");
        foreach (var child in _children)
            child.Display(indent + 2);
    }
}

// Usage — treats file and folder the same way
var root = new Folder("root");
var src = new Folder("src");
src.Add(new File("Program.cs", 12));
src.Add(new File("Startup.cs", 8));
root.Add(src);
root.Add(new File("README.md", 2));
root.Display(); // traverses entire tree
Console.WriteLine($"Total: {root.GetSize()}KB");
```

---

### 9. Decorator
**Also known as:** Wrapper

**Intent:** Attach new behaviors to objects at runtime by placing them inside wrapper objects that implement the same interface.

**Key insight:** Composition over inheritance. Each decorator wraps the previous one — stackable behaviors.

```csharp
// Component interface
public interface IDataProcessor
{
    string Process(string data);
}

// Concrete component
public class DataProcessor : IDataProcessor
{
    public string Process(string data) => data;
}

// Base decorator
public abstract class DataProcessorDecorator(IDataProcessor inner) : IDataProcessor
{
    protected readonly IDataProcessor _inner = inner;
    public abstract string Process(string data);
}

// Concrete decorators — each adds one behavior
public class CompressionDecorator(IDataProcessor inner) : DataProcessorDecorator(inner)
{
    public override string Process(string data)
    {
        var processed = _inner.Process(data);
        return $"COMPRESSED({processed})"; // simulate compression
    }
}

public class EncryptionDecorator(IDataProcessor inner) : DataProcessorDecorator(inner)
{
    public override string Process(string data)
    {
        var processed = _inner.Process(data);
        return $"ENCRYPTED({processed})"; // simulate encryption
    }
}

public class LoggingDecorator(IDataProcessor inner, ILogger logger) : DataProcessorDecorator(inner)
{
    public override string Process(string data)
    {
        logger.LogInformation("Processing data of length {Length}", data.Length);
        var result = _inner.Process(data);
        logger.LogInformation("Processing complete");
        return result;
    }
}

// Stack decorators — order matters!
IDataProcessor processor = new DataProcessor();
processor = new CompressionDecorator(processor);   // compress first
processor = new EncryptionDecorator(processor);    // then encrypt
processor = new LoggingDecorator(processor, logger); // log around everything

var result = processor.Process("Hello World");
// Logs → encrypts(compresses("Hello World"))

// Real-world .NET: ASP.NET Core Middleware IS the Decorator pattern
// app.UseAuthentication() wraps app.UseAuthorization() wraps ... 
```

---

### 10. Facade
**Intent:** Provide a simplified interface to a complex subsystem.

**Problem:** Working with a complex library or framework requires understanding many classes and their correct interaction order.

```csharp
// Complex subsystems
public class VideoEncoder     { public void Encode(string file, string format) { } }
public class AudioExtractor   { public byte[] Extract(string file) => []; }
public class ThumbnailGenerator { public string Generate(string file, int timestamp) => "thumb.jpg"; }
public class FileUploader     { public async Task<string> UploadAsync(string file, CancellationToken ct) => "url"; }
public class NotificationService { public void Notify(string userId, string message) { } }

// Facade — simple interface hiding all the complexity
public class VideoProcessingFacade(
    VideoEncoder encoder,
    AudioExtractor audio,
    ThumbnailGenerator thumbs,
    FileUploader uploader,
    NotificationService notifier)
{
    public async Task<VideoProcessingResult> ProcessAsync(
        string inputFile,
        string userId,
        CancellationToken ct = default)
    {
        // Client doesn't know any of this complexity
        encoder.Encode(inputFile, "mp4");
        var audioTrack = audio.Extract(inputFile);
        var thumbnail = thumbs.Generate(inputFile, 30);
        var url = await uploader.UploadAsync(inputFile, ct);
        notifier.Notify(userId, $"Your video is ready: {url}");

        return new VideoProcessingResult(url, thumbnail);
    }
}

// Client code — beautifully simple
var result = await videoFacade.ProcessAsync("raw-video.mov", userId, ct);
```

---

### 11. Flyweight
**Intent:** Fit more objects into available RAM by sharing common state between multiple objects instead of storing it in each.

**Problem:** Millions of small objects (particles, characters, trees) consume too much memory because each stores the same data.

**Solution:** Split object state into **intrinsic** (shared, immutable) and **extrinsic** (unique per instance, passed in).

```csharp
// Intrinsic state (shared) — stored in the Flyweight
public sealed class CharacterStyle(string font, int size, string color)
{
    public string Font { get; } = font;
    public int Size { get; } = size;
    public string Color { get; } = color;
}

// Flyweight factory — cache and reuse shared states
public class CharacterStyleFactory
{
    private readonly Dictionary<string, CharacterStyle> _cache = [];

    public CharacterStyle GetStyle(string font, int size, string color)
    {
        var key = $"{font}|{size}|{color}";
        if (!_cache.TryGetValue(key, out var style))
        {
            style = new CharacterStyle(font, size, color);
            _cache[key] = style;
        }
        return style;
    }
}

// Character holds extrinsic state (unique) + reference to shared flyweight
public class Character(char glyph, int x, int y, CharacterStyle style)
{
    public char Glyph { get; } = glyph;
    public int X { get; } = x;  // unique position
    public int Y { get; } = y;  // unique position
    public CharacterStyle Style { get; } = style; // shared style
}
```

---

### 12. Proxy
**Intent:** Provide a substitute or placeholder for another object that controls access to the original.

**Types:** Virtual proxy (lazy init), Protection proxy (access control), Remote proxy (network), Logging/Caching proxy.

```csharp
// Subject interface
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct = default);
}

// Real implementation
public class UserRepository(AppDbContext context) : IUserRepository
{
    public async Task<User?> GetByIdAsync(Guid id, CancellationToken ct) =>
        await context.Users.FirstOrDefaultAsync(u => u.Id == id, ct);
}

// Caching Proxy — transparent to client
public class CachedUserRepository(IUserRepository inner, IMemoryCache cache) : IUserRepository
{
    public async Task<User?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var key = $"user:{id}";
        if (cache.TryGetValue(key, out User? cached))
            return cached;

        var user = await inner.GetByIdAsync(id, ct);

        if (user is not null)
            cache.Set(key, user, TimeSpan.FromMinutes(5));

        return user;
    }
}

// Register proxy transparently
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.Decorate<IUserRepository, CachedUserRepository>(); // Scrutor library
```

---

## BEHAVIORAL PATTERNS

> _"These patterns are concerned with algorithms and the assignment of responsibilities between objects."_

---

### 13. Chain of Responsibility
**Intent:** Pass requests along a chain of handlers. Each handler decides to process or pass to next.

**Real-world .NET:** **ASP.NET Core Middleware pipeline**, MediatR pipeline behaviors, validation chains.

```csharp
// Handler interface
public abstract class ValidationHandler<T>
{
    private ValidationHandler<T>? _next;

    public ValidationHandler<T> SetNext(ValidationHandler<T> next)
    {
        _next = next;
        return next; // enables fluent chaining
    }

    public virtual ValidationResult Handle(T request)
    {
        if (_next is not null)
            return _next.Handle(request);

        return ValidationResult.Success();
    }
}

// Concrete handlers in the chain
public class NullCheckHandler : ValidationHandler<CreateUserRequest>
{
    public override ValidationResult Handle(CreateUserRequest req)
    {
        if (string.IsNullOrWhiteSpace(req.Name))
            return ValidationResult.Failure("Name is required");
        if (string.IsNullOrWhiteSpace(req.Email))
            return ValidationResult.Failure("Email is required");
        return base.Handle(req); // pass to next
    }
}

public class EmailFormatHandler : ValidationHandler<CreateUserRequest>
{
    public override ValidationResult Handle(CreateUserRequest req)
    {
        if (!req.Email.Contains('@'))
            return ValidationResult.Failure("Invalid email format");
        return base.Handle(req);
    }
}

public class PasswordStrengthHandler : ValidationHandler<CreateUserRequest>
{
    public override ValidationResult Handle(CreateUserRequest req)
    {
        if (req.Password.Length < 8)
            return ValidationResult.Failure("Password too short");
        return base.Handle(req);
    }
}

// Build the chain
var chain = new NullCheckHandler();
chain.SetNext(new EmailFormatHandler())
     .SetNext(new PasswordStrengthHandler());

var result = chain.Handle(request);

// Real-world: MediatR Pipeline Behavior IS Chain of Responsibility
// ValidationBehavior → LoggingBehavior → CachingBehavior → Handler
```

---

### 14. Command
**Also known as:** Action, Transaction

**Intent:** Turn a request into a stand-alone object containing all request information. Enables queuing, logging, and undo/redo.

**CQRS is a direct application of the Command pattern.**

```csharp
// Command object (encapsulates the request)
public record CreateOrderCommand(Guid CustomerId, List<OrderItem> Items) : IRequest<Result<Guid>>;

// Command handler (the "receiver" that does the work)
public class CreateOrderCommandHandler(
    IOrderRepository orders,
    IUnitOfWork uow,
    ILogger<CreateOrderCommandHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(
        CreateOrderCommand cmd, CancellationToken ct)
    {
        logger.LogInformation("Creating order for customer {CustomerId}", cmd.CustomerId);

        var order = Order.Create(cmd.CustomerId, cmd.Items);
        await orders.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);

        return Result.Success(order.Id);
    }
}

// Invoker — sends the command (MediatR is the invoker)
public class OrdersEndpoints
{
    public static async Task<IResult> CreateOrder(
        CreateOrderRequest req, ISender mediator, CancellationToken ct)
    {
        var cmd = new CreateOrderCommand(req.CustomerId, req.Items);
        var result = await mediator.Send(cmd, ct);
        return result.Match(
            id => Results.Created($"/orders/{id}", new { id }),
            errors => Results.Problem(errors.ToProblemDetails()));
    }
}
```

---

### 15. Iterator
**Intent:** Provide a way to traverse a collection without exposing its underlying representation.

**In .NET:** `IEnumerable<T>` / `IEnumerator<T>` IS the Iterator pattern. `yield return` creates iterators.

```csharp
// .NET's iterator pattern built-in
public class UserCollection : IEnumerable<User>
{
    private readonly List<User> _users = [];

    public IEnumerator<User> GetEnumerator() => _users.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

// Custom iterator with yield (lazy evaluation)
public static async IAsyncEnumerable<Product> StreamProductsAsync(
    IMongoCollection<Product> collection,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var cursor = await collection.FindAsync(_ => true, cancellationToken: ct);
    while (await cursor.MoveNextAsync(ct))
        foreach (var product in cursor.Current)
            yield return product;  // lazy — doesn't load all at once
}
```

---

### 16. Mediator
**Also known as:** Intermediary, Controller

**Intent:** Reduce chaotic dependencies between objects by routing all communication through a mediator object.

**Real-world .NET:** **MediatR library** is the Mediator pattern. Objects send commands/queries to the mediator, not to each other.

```csharp
// Without Mediator: OrderService calls UserService calls InventoryService calls...
// — tight coupling, spaghetti dependencies

// With Mediator (MediatR):
// Each handler only knows about its own command/query — fully decoupled

// Registration
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

// Command side (write)
await _mediator.Send(new CreateOrderCommand(customerId, items));

// Query side (read)
var order = await _mediator.Send(new GetOrderByIdQuery(orderId));

// Events (many handlers can respond)
await _mediator.Publish(new OrderCreatedEvent(order.Id));
// → EmailNotificationHandler, InventoryUpdateHandler, AuditHandler all fire
```

---

### 17. Memento
**Also known as:** Snapshot

**Intent:** Save and restore the previous state of an object without revealing its implementation details.

**Use for:** Undo/redo, snapshots, state history.

```csharp
// Memento — snapshot of state (immutable record)
public sealed record EditorMemento(string Content, int CursorPosition, DateTimeOffset SavedAt);

// Originator — creates and restores from mementos
public class TextEditor
{
    private string _content = string.Empty;
    private int _cursorPosition = 0;

    public void Type(string text)
    {
        _content = _content.Insert(_cursorPosition, text);
        _cursorPosition += text.Length;
    }

    public EditorMemento Save() =>
        new(_content, _cursorPosition, DateTimeOffset.UtcNow);

    public void Restore(EditorMemento memento)
    {
        _content = memento.Content;
        _cursorPosition = memento.CursorPosition;
    }
}

// Caretaker — manages the history
public class UndoManager
{
    private readonly Stack<EditorMemento> _history = new();
    private readonly TextEditor _editor;

    public UndoManager(TextEditor editor) => _editor = editor;

    public void SaveState() => _history.Push(_editor.Save());

    public void Undo()
    {
        if (_history.Count > 0)
            _editor.Restore(_history.Pop());
    }
}
```

---

### 18. Observer
**Also known as:** Event-Subscriber, Listener

**Intent:** Define a subscription mechanism to notify multiple objects when an event occurs.

**Real-world .NET:** C# `event`/`EventHandler`, `INotifyPropertyChanged`, `IObservable<T>`/`IObserver<T>`, MediatR `INotification`.

```csharp
// Publisher (Subject)
public class OrderService
{
    // C# event IS the Observer pattern
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest req, CancellationToken ct)
    {
        var order = Order.Create(req.CustomerId, req.Items);
        // ... save to DB ...

        // Notify all subscribers
        OrderCreated?.Invoke(this, new OrderCreatedEventArgs(order));
        return order;
    }
}

// Subscribers
public class EmailNotificationHandler
{
    public void Subscribe(OrderService service) =>
        service.OrderCreated += async (_, e) =>
            await SendOrderConfirmationEmailAsync(e.Order);
}

// Enterprise approach: Domain Events via MediatR (decoupled)
public record OrderCreatedEvent(Guid OrderId, Guid CustomerId) : INotification;

// Multiple independent handlers — no coupling between them
public class SendConfirmationEmailHandler : INotificationHandler<OrderCreatedEvent> { }
public class UpdateInventoryHandler : INotificationHandler<OrderCreatedEvent> { }
public class CreateAuditLogHandler : INotificationHandler<OrderCreatedEvent> { }
```

---

### 19. State
**Intent:** Let an object alter its behavior when its internal state changes. Appears as if the object changed its class.

**Use for:** Order status machines, connection states, workflow engines.

```csharp
// State interface
public interface IOrderState
{
    void Pay(Order order);
    void Ship(Order order);
    void Deliver(Order order);
    void Cancel(Order order);
    string StatusName { get; }
}

// Concrete states
public class PendingState : IOrderState
{
    public string StatusName => "Pending";
    public void Pay(Order order)    => order.TransitionTo(new PaidState());
    public void Ship(Order order)   => throw new InvalidOperationException("Pay first");
    public void Deliver(Order order)=> throw new InvalidOperationException("Pay first");
    public void Cancel(Order order) => order.TransitionTo(new CancelledState());
}

public class PaidState : IOrderState
{
    public string StatusName => "Paid";
    public void Pay(Order order)    => throw new InvalidOperationException("Already paid");
    public void Ship(Order order)   => order.TransitionTo(new ShippedState());
    public void Deliver(Order order)=> throw new InvalidOperationException("Ship first");
    public void Cancel(Order order) => order.TransitionTo(new CancelledState());
}

// Context — delegates to current state
public class Order
{
    private IOrderState _state = new PendingState();

    internal void TransitionTo(IOrderState newState)
    {
        Console.WriteLine($"Order {Id}: {_state.StatusName} → {newState.StatusName}");
        _state = newState;
    }

    public void Pay()     => _state.Pay(this);
    public void Ship()    => _state.Ship(this);
    public void Deliver() => _state.Deliver(this);
    public void Cancel()  => _state.Cancel(this);
    public string Status  => _state.StatusName;
    public Guid Id        { get; init; } = Guid.NewGuid();
}
```

---

### 20. Strategy
**Intent:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Real-world .NET:** Sorting strategies, payment processors, discount calculators, route planners.

```csharp
// Strategy interface
public interface IDiscountStrategy
{
    decimal Calculate(decimal originalPrice, Customer customer);
}

// Concrete strategies
public class NoDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal price, Customer _) => price;
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal price, Customer c) =>
        c.LoyaltyYears >= 5 ? price * 0.85m : price * 0.95m;
}

public class SeasonalDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal price, Customer _) =>
        DateTime.UtcNow.Month == 12 ? price * 0.80m : price;
}

public class BulkDiscount : IDiscountStrategy
{
    private readonly int _minQuantity;
    public BulkDiscount(int minQuantity) => _minQuantity = minQuantity;
    public decimal Calculate(decimal price, Customer _) =>
        price * 0.75m; // assumes context has quantity check
}

// Context — uses a strategy
public class OrderPricer(IDiscountStrategy discountStrategy)
{
    public decimal GetPrice(decimal basePrice, Customer customer) =>
        discountStrategy.Calculate(basePrice, customer);
}

// Select strategy at runtime (e.g., from DI or factory)
var strategy = customer.IsPremium
    ? new LoyaltyDiscount()
    : new NoDiscount();

var pricer = new OrderPricer(strategy);
var finalPrice = pricer.GetPrice(99.99m, customer);
```

---

### 21. Template Method
**Intent:** Define the skeleton of an algorithm in a base class, deferring some steps to subclasses.

```csharp
// Abstract class defines the template
public abstract class DataImporter
{
    // Template method — defines the algorithm
    public async Task ImportAsync(string source, CancellationToken ct = default)
    {
        var rawData = await ReadDataAsync(source, ct);    // step 1 (abstract)
        var validated = ValidateData(rawData);            // step 2 (abstract)
        var transformed = TransformData(validated);       // step 3 (can override)
        await SaveDataAsync(transformed, ct);             // step 4 (abstract)
        await OnImportCompleteAsync(ct);                  // step 5 (hook - optional)
    }

    protected abstract Task<string> ReadDataAsync(string source, CancellationToken ct);
    protected abstract IEnumerable<Record> ValidateData(string raw);
    protected abstract IEnumerable<Record> TransformData(IEnumerable<Record> records);
    protected abstract Task SaveDataAsync(IEnumerable<Record> records, CancellationToken ct);

    // Hook — subclasses can override optionally
    protected virtual Task OnImportCompleteAsync(CancellationToken ct) => Task.CompletedTask;
}

// Concrete implementations — fill in the steps
public class CsvImporter(IUserRepository repo) : DataImporter
{
    protected override Task<string> ReadDataAsync(string path, CancellationToken ct) =>
        File.ReadAllTextAsync(path, ct);

    protected override IEnumerable<Record> ValidateData(string raw) =>
        raw.Split('\n').Skip(1).Where(l => !string.IsNullOrWhiteSpace(l))
           .Select(l => Record.FromCsvLine(l));

    protected override IEnumerable<Record> TransformData(IEnumerable<Record> records) =>
        records.Select(r => r.Normalize());

    protected override Task SaveDataAsync(IEnumerable<Record> records, CancellationToken ct) =>
        repo.BulkInsertAsync(records, ct);
}
```

---

### 22. Visitor
**Intent:** Separate an algorithm from the objects on which it operates. Add new operations without modifying the classes.

```csharp
// Element interface
public interface IDocumentElement
{
    void Accept(IDocumentVisitor visitor);
}

// Concrete elements
public class Paragraph(string text) : IDocumentElement
{
    public string Text => text;
    public void Accept(IDocumentVisitor visitor) => visitor.VisitParagraph(this);
}

public class Image(string url, int width, int height) : IDocumentElement
{
    public string Url => url;
    public int Width => width;
    public int Height => height;
    public void Accept(IDocumentVisitor visitor) => visitor.VisitImage(this);
}

// Visitor interface
public interface IDocumentVisitor
{
    void VisitParagraph(Paragraph p);
    void VisitImage(Image img);
}

// Concrete visitors — add operations without touching elements
public class HtmlRenderer : IDocumentVisitor
{
    private readonly StringBuilder _html = new();
    public void VisitParagraph(Paragraph p) => _html.Append($"<p>{p.Text}</p>");
    public void VisitImage(Image img) => _html.Append($"<img src='{img.Url}' width='{img.Width}'/>");
    public string GetHtml() => _html.ToString();
}

public class WordCountVisitor : IDocumentVisitor
{
    public int Count { get; private set; }
    public void VisitParagraph(Paragraph p) => Count += p.Text.Split(' ').Length;
    public void VisitImage(Image _) { } // images don't have words
}
```

---

### 23. Interpreter *(less commonly used)*
**Intent:** Define a grammar for a language and provide an interpreter for it.

**Use for:** SQL parsers, expression evaluators, rule engines, regex engines.

---

## PATTERN CHEAT SHEET

### Creational (5)
| Pattern | One-liner | When |
|---|---|---|
| **Factory Method** | Subclass decides which class to create | Multiple implementations of one thing |
| **Abstract Factory** | Creates families of related objects | Multiple related implementations must be used together |
| **Builder** | Step-by-step construction, fluent API | Complex objects with many optional parts |
| **Prototype** | Clone existing objects | Expensive to create from scratch, small variations needed |
| **Singleton** | One instance globally | Config, logger, connection pool — prefer DI |

### Structural (7)
| Pattern | One-liner | When |
|---|---|---|
| **Adapter** | Make incompatible interfaces work together | Integrating third-party/legacy code |
| **Bridge** | Separate abstraction from implementation | Two independent dimensions of variation |
| **Composite** | Treat trees uniformly (leaf = branch) | File systems, UI trees, org charts |
| **Decorator** | Stack behaviors at runtime (wrap → wrap → wrap) | Logging, caching, compression, auth around existing code |
| **Facade** | Simple interface to complex subsystem | Complex libraries, SDK wrappers |
| **Flyweight** | Share common state to reduce memory | Millions of small objects with shared data |
| **Proxy** | Control access to another object | Lazy loading, caching, access control, remote objects |

### Behavioral (11)
| Pattern | One-liner | When |
|---|---|---|
| **Chain of Responsibility** | Pass request through handler chain | Middleware, validation pipelines |
| **Command** | Request as an object (undo, queue, log) | CQRS, undo/redo, task queues |
| **Iterator** | Traverse without exposing internals | Collections, generators, async streams |
| **Mediator** | Route all comms through a hub | Decoupling many classes, MediatR |
| **Memento** | Save/restore object state | Undo/redo, snapshots, autosave |
| **Observer** | Notify many on state change | Event systems, pub/sub, reactive |
| **State** | Change behavior based on internal state | State machines, workflows, order status |
| **Strategy** | Swap algorithm at runtime | Discount engines, payment processors, sorting |
| **Template Method** | Skeleton algorithm, steps overridable | Import pipelines, report generators |
| **Visitor** | Add operations without modifying classes | Document rendering, AST traversal |
| **Interpreter** | Parse and execute a grammar | SQL, expressions, rule engines |

---

## PATTERNS IN .NET 10 + ASP.NET CORE

| Pattern | .NET / ASP.NET Core Expression |
|---|---|
| Factory Method | `IHttpClientFactory`, `ILoggerFactory`, `DbProviderFactory` |
| Abstract Factory | Provider model (EF Core providers, auth providers) |
| Builder | `WebApplicationBuilder`, `IHostBuilder`, `DbContextOptionsBuilder` |
| Singleton | `AddSingleton<T>()` in DI |
| Decorator | ASP.NET Core Middleware pipeline; Scrutor's `.Decorate<T>()` |
| Facade | `WebApplication` hides Kestrel + routing + DI complexity |
| Proxy | `IMemoryCache` wrapping repositories; YARP reverse proxy |
| Chain of Responsibility | ASP.NET Core Middleware; MediatR pipeline behaviors |
| Command | MediatR `IRequest<T>` + `IRequestHandler<T>` (CQRS Commands) |
| Mediator | MediatR `ISender`/`IPublisher` |
| Observer | C# `event`, `IObservable<T>`, MediatR `INotification` |
| Strategy | `IDiscountStrategy`, `IPaymentProcessor`, `IAuthorizationHandler` |
| Template Method | `BackgroundService.ExecuteAsync()`, `DbContext.OnModelCreating()` |
| Iterator | `IEnumerable<T>`, `IAsyncEnumerable<T>`, `yield return` |
| State | Workflow engines, order state machines |

---

## SOLID PRINCIPLES (GoF Foundations)

| Principle | Rule | Pattern Support |
|---|---|---|
| **S** — Single Responsibility | One class, one reason to change | Command, Strategy, Template Method |
| **O** — Open/Closed | Open for extension, closed for modification | Strategy, Decorator, Visitor |
| **L** — Liskov Substitution | Subtypes must be substitutable for base types | Factory Method, Template Method |
| **I** — Interface Segregation | Many small interfaces > one fat interface | All patterns using interfaces |
| **D** — Dependency Inversion | Depend on abstractions, not concretions | All creational patterns, Decorator, Proxy |

---

_Sources: refactoring.guru/design-patterns, GoF "Design Patterns" book (Gamma, Helm, Johnson, Vlissides)_
