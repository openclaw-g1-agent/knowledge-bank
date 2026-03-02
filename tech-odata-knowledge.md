# OData — Complete Knowledge Base
_Learned: 2026-03-01 | Sources: odata.org/documentation, odata.org/getting-started, learn.microsoft.com/odata/webapi-8_

---

## What Is OData?

**OData (Open Data Protocol)** is an **ISO/IEC-approved, OASIS-standardized** data access protocol built on HTTP and REST principles. It defines:
- Standard URL conventions for querying and manipulating data
- A JSON-based payload format
- A metadata/schema description language (CSDL)
- Best practices for consistent, strongly-typed REST APIs

> Think of OData as **"SQL over HTTP"** — clients can filter, sort, paginate, project, expand, and aggregate server-side data using URL query parameters, without the server writing custom endpoint logic for each case.

**Current version:** OData V4.01 (OASIS Committee Specification, ISO/IEC approved)

---

## Why OData? Key Benefits

| Benefit | Description |
|---------|-------------|
| **Self-describing** | `$metadata` endpoint exposes the full schema — types, endpoints, capabilities |
| **Flexible querying** | Clients control filtering, sorting, paging, field selection — no custom endpoints |
| **Interoperability** | Language-agnostic; clients can be .NET, JS, Python, Power BI, Excel, SAP |
| **Standards-based** | ISO/IEC approved — widely supported by Microsoft tooling, SAP, Salesforce |
| **Client code gen** | OData CLI + Connected Service auto-generate typed client proxies from `$metadata` |
| **Reduces API surface** | One endpoint per entity replaces dozens of REST endpoints |
| **Analytics-friendly** | `$apply` aggregation extension enables server-side groupby, sum, count, avg |
| **Batch support** | Multiple operations in a single HTTP request |

---

## OData vs REST vs GraphQL

| Feature | REST | OData | GraphQL |
|---------|------|-------|---------|
| **Querying** | Fixed endpoints | URL query params (`$filter`, `$select`…) | Client-defined query document |
| **Schema** | OpenAPI/Swagger | CSDL (`$metadata`) | SDL (Schema Definition Language) |
| **Filtering** | Manual, per endpoint | Built-in (`$filter`) | Via arguments |
| **Pagination** | Manual | Built-in (`$top`, `$skip`, `$skiptoken`) | Relay Connection spec |
| **Relations** | Multiple calls | `$expand` | Nested fields |
| **Aggregation** | Manual | `$apply` extension | N/A (use separate endpoint) |
| **Real-time** | Polling | Polling | Subscriptions (WebSocket) |
| **Tooling (.NET)** | Minimal API / MVC | `Microsoft.AspNetCore.OData` | Hot Chocolate |
| **Best for** | Simple APIs | Data-heavy, analytics, enterprise integrations | Flexible client-driven APIs |

---

## OData URL Anatomy

```
https://api.example.com/odata/Properties?$filter=Price gt 500000&$select=Address,Price&$top=10
|___________________________|___________|__________________________________________|___________|
        Service Root          Entity Set              Query Options                   Pagination
```

---

## System Query Options — Complete Reference

### `$filter` — Filter Collections

The most powerful query option. Evaluates a Boolean expression on each resource; returns only matching items.

#### Comparison Operators
```
eq   Equal                   /Properties?$filter=Status eq 'Active'
ne   Not equal               /Properties?$filter=Status ne 'Sold'
gt   Greater than            /Properties?$filter=Price gt 500000
ge   Greater than or equal   /Properties?$filter=Bedrooms ge 3
lt   Less than               /Properties?$filter=Price lt 1000000
le   Less than or equal      /Properties?$filter=LotSize le 5000
```

#### Logical Operators
```
and  /Properties?$filter=Price gt 500000 and Bedrooms ge 3
or   /Properties?$filter=Status eq 'Auction' or Status eq 'Active'
not  /Properties?$filter=not(Status eq 'Sold')
```

#### Arithmetic Operators
```
add, sub, mul, div, mod
/Properties?$filter=Price add TaxAmount gt 600000
```

#### String Functions
```
contains(field, 'value')     /Properties?$filter=contains(Address, 'Austin')
startswith(field, 'value')   /Properties?$filter=startswith(ZipCode, '787')
endswith(field, 'value')     /Properties?$filter=endswith(Email, '@realty.com')
tolower(field)               /Properties?$filter=tolower(City) eq 'austin'
toupper(field)               /Properties?$filter=toupper(Status) eq 'ACTIVE'
trim(field)
concat(field1, field2)
indexof(field, 'value')
substring(field, start)
length(field)
```

#### Date/Time Functions
```
year(field), month(field), day(field)
hour(field), minute(field), second(field)
now(), maxdatetime(), mindatetime()
date(field), time(field)

/Auctions?$filter=year(StartDate) eq 2026
/Auctions?$filter=StartDate gt now()
```

#### Math Functions
```
round(field), floor(field), ceiling(field)
/Properties?$filter=round(Price div 1000) gt 500
```

#### Type Functions
```
isof(TypeName)           — check entity type (for inheritance)
cast(field, TypeName)    — cast to derived type
```

#### Filter on Complex Types
```
/Properties?$filter=contains(Location/Address, 'Lake')
/Properties?$filter=Location/City eq 'Austin'
```

#### Filter on Enum Properties
```
/Properties?$filter=PropertyType eq 'MyNamespace.PropertyType''Residential'
```

#### Nested Filter in $expand
```
/Sellers?$expand=Listings($filter=Status eq 'Active')
```

#### Lambda Operators — Filter on Collections
```
any(alias:condition)   — true if ANY element satisfies condition
all(alias:condition)   — true if ALL elements satisfy condition

# Properties with any photo tagged 'exterior'
/Properties?$filter=Photos/any(p:p/Tag eq 'exterior')

# Auctions where ALL bids exceed reserve
/Auctions?$filter=Bids/all(b:b/Amount gt ReservePrice)

# Nested lambda (collection of complex type)
/Buyers?$filter=SavedSearches/any(s:s/City eq 'Austin' and s/MinBedrooms ge 3)
```

---

### `$select` — Field Projection

Returns only specified fields — reduces payload size significantly:
```
GET /Properties?$select=Id,Address,Price,Status
GET /Properties?$select=Address,Location/City,Agent/Name
```

Combined with `$expand`:
```
GET /Properties?$select=Address,Price&$expand=Agent($select=Name,Phone)
```

---

### `$expand` — Include Related Data (JOIN)

Traverses navigation properties to include related entities inline:
```
GET /Properties?$expand=Agent
GET /Properties?$expand=Agent,Photos
GET /Properties?$expand=Agent($select=Name,Email)
GET /Auctions?$expand=Property($expand=Agent)           ← nested expand
GET /Properties?$expand=Bids($orderby=Amount desc;$top=3)  ← expand with options
```

Combine select + expand + filter:
```
GET /Properties?$select=Address,Price&$expand=Bids($filter=Amount gt 500000;$select=Amount,BidderName;$orderby=Amount desc)
```

---

### `$orderby` — Sorting

```
GET /Properties?$orderby=Price                        ← ascending (default)
GET /Properties?$orderby=Price desc
GET /Properties?$orderby=City asc,Price desc          ← multi-field sort
GET /Properties?$orderby=Agent/Name asc              ← sort by navigation property
```

---

### `$top` and `$skip` — Offset Pagination

```
GET /Properties?$top=10              ← first 10
GET /Properties?$top=10&$skip=20     ← page 3 (items 21-30)
GET /Properties?$top=10&$skip=0&$orderby=Price  ← always orderby for stable paging
```

⚠️ **Offset pagination limitations:**
- Unstable when data changes between pages
- For large datasets, use server-driven paging with `$skiptoken` instead

#### Server-Driven Pagination (`$skiptoken`)
When server enforces page limits, the response includes `@odata.nextLink`:
```json
{
  "@odata.context": "serviceRoot/$metadata#Properties",
  "@odata.nextLink": "serviceRoot/Properties?$skiptoken=abc123xyz",
  "value": [...]
}
```
Client follows `nextLink` to get the next page — cursor-based, stable.

---

### `$count` — Total Record Count

```
GET /Properties?$count=true
```
Response includes `@odata.count` at the top level:
```json
{
  "@odata.count": 1423,
  "value": [...]
}
```

Standalone count:
```
GET /Properties/$count       → returns plain integer: 1423
```

---

### `$search` — Full-Text Search

```
GET /Properties?$search=waterfront
GET /Properties?$search=Austin AND luxury
GET /Properties?$search="gated community"
```
Implementation-defined — server decides what "match" means (full-text index, contains, etc.)

---

### `$apply` — Aggregation (OData Extension)

Powerful analytics without custom endpoints:
```
# Count by status
GET /Properties?$apply=groupby((Status),aggregate($count as Count))

# Average price by city
GET /Properties?$apply=groupby((City),aggregate(Price with average as AvgPrice))

# Total auction revenue
GET /Auctions?$apply=aggregate(WinningBid with sum as TotalRevenue)

# Combine: filter → group → aggregate
GET /Properties?$apply=filter(Status eq 'Sold')/groupby((City,PropertyType),aggregate(Price with average as AvgSalePrice,Price with max as MaxSalePrice))
```

Aggregation functions: `sum`, `min`, `max`, `average`, `countdistinct`, `$count`

---

## OData Data Modification

### Create an Entity
```http
POST /Properties
OData-Version: 4.0
Content-Type: application/json;odata.metadata=minimal

{
  "@odata.type": "MyNamespace.Property",
  "Address": "1234 Oak Lane",
  "Price": 750000,
  "Status": "Active",
  "Bedrooms": 4
}
```
Response: `201 Created` with `Location` header and created entity body.

### Update an Entity (PATCH — Preferred)
```http
PATCH /Properties('prop_123')
OData-Version: 4.0
Content-Type: application/json;odata.metadata=minimal

{
  "@odata.type": "MyNamespace.Property",
  "Price": 725000,
  "Status": "UnderContract"
}
```
Response: `204 No Content` (only changed fields are updated)

### Full Replace (PUT — less common)
```http
PUT /Properties('prop_123')
```
Replaces the entire entity — all unspecified fields reset to defaults.

### Delete an Entity
```http
DELETE /Properties('prop_123')
```
Response: `204 No Content`

### ETag — Optimistic Concurrency
GET returns ETag in response header:
```http
@odata.etag: W/"08D1694BF26D2BC9"
```

Include `If-Match` on update/delete to prevent lost updates:
```http
PATCH /Properties('prop_123')
If-Match: W/"08D1694BF26D2BC9"

{ "Price": 710000 }
```
If another client updated first → `412 Precondition Failed`

### Manage Relationships (`$ref`)
```http
# Set auction property link
PUT /Auctions('auc_1')/Property/$ref
{ "@odata.id": "serviceRoot/Properties('prop_123')" }

# Add to collection (many-to-many)
POST /Buyers('buyer_1')/SavedProperties/$ref
{ "@odata.id": "serviceRoot/Properties('prop_456')" }

# Remove from collection
DELETE /Buyers('buyer_1')/SavedProperties('prop_456')/$ref
```

---

## OData Metadata Endpoint

Every OData service exposes a machine-readable schema:
```
GET /odata/$metadata
```
Returns CSDL (Common Schema Definition Language) XML — describes:
- All entity types and their properties + types
- Navigation properties (relationships)
- Actions and functions
- Entity sets and singletons
- Enum types, complex types

Used by:
- OData Connected Service → auto-generates typed C# client
- OData CLI → generates client proxies
- Power BI → directly connects to OData feeds
- Excel → imports OData as a live data source

---

## OData Functions and Actions

### Functions — Read-Only, No Side Effects (GET)
```http
# Unbound function
GET /GetPropertiesNearPoint(lat=30.2672,lon=-97.7431,radiusMiles=5)

# Bound function (bound to entity type)
GET /Properties('prop_123')/GetSimilarListings(maxResults=5)
GET /Sellers('seller_1')/GetTotalRevenue()
```

### Actions — May Have Side Effects (POST)
```http
# Bound action on entity
POST /Auctions('auc_1')/PlaceBid
{ "amount": 750000, "bidderId": "buyer_123" }

# Bound action on collection
POST /Properties/BulkUpdateStatus
{ "ids": ["prop_1","prop_2"], "status": "Archived" }
```

---

## Advanced OData Features

### Singletons
A single named entity (not a collection):
```
GET /Me                    ← current user singleton
GET /Me/SavedProperties    ← properties of singleton
PATCH /Me                  ← update singleton
```

### Inheritance / Derived Types
```
GET /Listings('id')/MyNamespace.ResidentialListing      ← get as derived type
GET /Listings/MyNamespace.CommercialListing              ← get derived collection
GET /Listings?$filter=MyNamespace.ResidentialListing/Bedrooms ge 3  ← filter on derived
POST /Listings { "@odata.type": "#MyNamespace.AuctionListing", ... }  ← create derived
```

### Containment Navigation Properties
Entities that only exist as part of another entity:
```
GET /Properties('prop_1')/Rooms              ← rooms are contained in property
POST /Properties('prop_1')/Rooms             ← create room (contained)
DELETE /Properties('prop_1')/Rooms(3)        ← delete contained entity
```

### Open Types
Entity types that accept additional undeclared properties at runtime:
```http
POST /Properties
{
  "Address": "123 Main",
  "Price": 500000,
  "CustomField_PoolType": "Infinity"   ← undeclared, accepted by open type
}
```

### Batch Requests
Group multiple operations into a single HTTP call:
```http
POST /odata/$batch
Content-Type: multipart/mixed;boundary=batch_abc123

--batch_abc123
Content-Type: application/http
GET /Properties HTTP/1.1

--batch_abc123
Content-Type: application/http
Content-ID: 1
POST /Properties HTTP/1.1
Content-Type: application/json
{ "Address": "New Property", "Price": 500000 }

--batch_abc123
Content-Type: application/http
GET /Properties HTTP/1.1
--batch_abc123--
```
Response: single multipart response containing all individual responses.

JSON batch format (OData V4.01):
```json
POST /odata/$batch
Content-Type: application/json

{
  "requests": [
    { "id": "1", "method": "GET", "url": "Properties" },
    { "id": "2", "method": "POST", "url": "Properties", "body": { "Address": "..." } },
    { "id": "3", "method": "GET", "url": "Properties", "dependsOn": ["2"] }
  ]
}
```

---

## OData Response Format

### Standard Response
```json
{
  "@odata.context": "https://api.example.com/odata/$metadata#Properties",
  "@odata.count": 1423,
  "@odata.nextLink": "https://api.example.com/odata/Properties?$skiptoken=abc",
  "value": [
    {
      "@odata.id": "https://api.example.com/odata/Properties('prop_1')",
      "@odata.etag": "W/\"abc123\"",
      "@odata.editLink": "https://api.example.com/odata/Properties('prop_1')",
      "Id": "prop_1",
      "Address": "123 Main St",
      "Price": 750000
    }
  ]
}
```

### Key Annotations
| Annotation | Purpose |
|-----------|---------|
| `@odata.context` | URL of the metadata for this response |
| `@odata.count` | Total count when `$count=true` |
| `@odata.nextLink` | URL of next page (server-driven paging) |
| `@odata.id` | Canonical URL of the entity |
| `@odata.etag` | ETag for optimistic concurrency |
| `@odata.editLink` | URL for editing (if different from id) |
| `@odata.type` | Type name (for polymorphic/derived types) |

---

## Implementation in ASP.NET Core (.NET 8/9/10)

### NuGet Package
```xml
<PackageReference Include="Microsoft.AspNetCore.OData" Version="8.*" />
<PackageReference Include="Microsoft.OData.ModelBuilder" Version="1.*" />
```

### Program.cs Setup
```csharp
var builder = WebApplication.CreateBuilder(args);

// Build the EDM (Entity Data Model)
var edmModel = GetEdmModel();

builder.Services.AddControllers()
    .AddOData(options => options
        .AddRouteComponents("odata", edmModel)
        .Select()      // enable $select
        .Filter()      // enable $filter
        .OrderBy()     // enable $orderby
        .Expand()      // enable $expand
        .Count()       // enable $count
        .SetMaxTop(100) // max page size
    );

var app = builder.Build();
app.MapControllers();
app.Run();

// EDM Model Builder
static IEdmModel GetEdmModel()
{
    var builder = new ODataConventionModelBuilder();
    builder.EntitySet<Property>("Properties");
    builder.EntitySet<Auction>("Auctions");
    builder.EntitySet<Bid>("Bids");
    builder.EntitySet<Agent>("Agents");
    builder.EntitySet<Buyer>("Buyers");
    builder.Singleton<CurrentUser>("Me");
    return builder.GetEdmModel();
}
```

### OData Controller
```csharp
[Route("odata")]
public class PropertiesController : ODataController
{
    private readonly IPropertyRepository _repo;

    public PropertiesController(IPropertyRepository repo) => _repo = repo;

    // GET /odata/Properties
    // Supports: $filter, $select, $expand, $orderby, $top, $skip, $count
    [HttpGet]
    [EnableQuery(MaxTop = 100, AllowedQueryOptions = AllowedQueryOptions.All)]
    public IQueryable<Property> Get()
    {
        return _repo.GetAll();   // return IQueryable — OData translates to DB query
    }

    // GET /odata/Properties('prop_123')
    [HttpGet("{key}")]
    [EnableQuery]
    public SingleResult<Property> Get([FromRoute] string key)
    {
        return SingleResult.Create(_repo.GetById(key));
    }

    // POST /odata/Properties
    [HttpPost]
    public async Task<IActionResult> Post([FromBody] Property property)
    {
        if (!ModelState.IsValid) return BadRequest(ModelState);
        await _repo.AddAsync(property);
        return Created(property);
    }

    // PATCH /odata/Properties('prop_123')
    [HttpPatch("{key}")]
    public async Task<IActionResult> Patch([FromRoute] string key, [FromBody] Delta<Property> delta)
    {
        var property = await _repo.GetByIdAsync(key);
        if (property is null) return NotFound();
        delta.Patch(property);  // applies only changed fields
        await _repo.UpdateAsync(property);
        return Updated(property);
    }

    // DELETE /odata/Properties('prop_123')
    [HttpDelete("{key}")]
    public async Task<IActionResult> Delete([FromRoute] string key)
    {
        var property = await _repo.GetByIdAsync(key);
        if (property is null) return NotFound();
        await _repo.DeleteAsync(property);
        return NoContent();
    }
}
```

### `[EnableQuery]` Attribute — The Core Attribute
```csharp
[EnableQuery(
    MaxTop = 100,                          // max $top value
    MaxSkip = 1000,                        // max $skip value
    MaxExpansionDepth = 3,                 // max $expand nesting
    MaxNodeCount = 20,                     // max nodes in $filter expression
    PageSize = 25,                         // server-driven page size (auto nextLink)
    AllowedQueryOptions = AllowedQueryOptions.Select
                        | AllowedQueryOptions.Filter
                        | AllowedQueryOptions.OrderBy
                        | AllowedQueryOptions.Expand
                        | AllowedQueryOptions.Count,
    AllowedFunctions = AllowedFunctions.AllStringFunctions,
    AllowedLogicalOperators = AllowedLogicalOperators.All,
    AllowedArithmeticOperators = AllowedArithmeticOperators.All
)]
```

### Returning `IQueryable<T>` — Critical for Performance
```csharp
// ✅ CORRECT — OData translates query options to LINQ → single optimized DB query
[EnableQuery]
public IQueryable<Property> Get()
{
    return _context.Properties.AsNoTracking();
    // OData applies: WHERE, ORDER BY, LIMIT, SELECT — all in SQL
}

// ❌ WRONG — loads all data into memory first, then filters in memory
[EnableQuery]
public IEnumerable<Property> Get()
{
    return _context.Properties.ToList();  // pulls ALL records, then OData filters
}
```

### EDM Fluent Configuration (Full Control)
```csharp
static IEdmModel GetEdmModel()
{
    var builder = new ODataConventionModelBuilder();

    // Entity sets
    var properties = builder.EntitySet<Property>("Properties");
    var auctions = builder.EntitySet<Auction>("Auctions");

    // Navigation property configuration
    properties.EntityType.HasMany(p => p.Photos);
    properties.EntityType.HasOptional(p => p.Agent);

    // Custom functions (bound)
    var getSimilar = properties.EntityType
        .Function("GetSimilarListings")
        .ReturnsCollectionFromEntitySet<Property>("Properties");
    getSimilar.Parameter<int>("maxResults");

    // Custom actions (bound)
    var placeBid = auctions.EntityType
        .Action("PlaceBid")
        .Returns<BidResult>();
    placeBid.Parameter<decimal>("amount");
    placeBid.Parameter<string>("bidderId");

    // Unbound functions
    var nearPoint = builder.Function("GetPropertiesNearPoint")
        .ReturnsCollectionFromEntitySet<Property>("Properties");
    nearPoint.Parameter<double>("lat");
    nearPoint.Parameter<double>("lon");
    nearPoint.Parameter<double>("radiusMiles");

    // Enum types
    builder.EnumType<PropertyStatus>();
    builder.EnumType<PropertyType>();

    // Singleton
    builder.Singleton<CurrentUser>("Me");

    return builder.GetEdmModel();
}
```

### Action/Function Handlers
```csharp
// Bound action handler
[HttpPost("PlaceBid")]
public async Task<IActionResult> PlaceBid([FromODataUri] string key, ODataActionParameters parameters)
{
    var amount = (decimal)parameters["amount"];
    var bidderId = (string)parameters["bidderId"];

    var auction = await _repo.GetAuctionAsync(key);
    var result = await _bidService.PlaceBidAsync(auction, amount, bidderId);

    return Ok(result);
}

// Unbound function handler
[HttpGet("GetPropertiesNearPoint(lat={lat},lon={lon},radiusMiles={radius})")]
[EnableQuery]
public IQueryable<Property> GetPropertiesNearPoint(double lat, double lon, double radius)
{
    return _context.Properties
        .Where(p => p.DistanceTo(lat, lon) <= radius)
        .AsNoTracking();
}
```

### OData with EF Core — Seamless Integration
```csharp
// IQueryable from EF Core DbContext is the ideal data source
// OData applies all query options directly to the SQL query

public class AppDbContext : DbContext
{
    public DbSet<Property> Properties => Set<Property>();
    public DbSet<Auction> Auctions => Set<Auction>();
    public DbSet<Bid> Bids => Set<Bid>();
}

// Controller — EF Core + OData = single optimized SQL per request
[EnableQuery]
public IQueryable<Property> Get() => _context.Properties.AsNoTracking();

// OData automatically generates SQL like:
// SELECT TOP 10 Id, Address, Price FROM Properties
// WHERE Status = 'Active' AND Price > 500000
// ORDER BY Price DESC
```

---

## Security — OData in Production

### Limit Query Capabilities
```csharp
// Only expose what you need
[EnableQuery(
    AllowedQueryOptions = AllowedQueryOptions.Filter
                        | AllowedQueryOptions.Select
                        | AllowedQueryOptions.OrderBy
                        | AllowedQueryOptions.Top
                        | AllowedQueryOptions.Skip
                        | AllowedQueryOptions.Count,
    MaxTop = 100,
    MaxSkip = 10000,
    MaxExpansionDepth = 2,
    MaxNodeCount = 15
)]
```

### Disable Dangerous Options Per Entity
```csharp
// On the model — disable expand for sensitive nav properties
properties.EntityType.HasMany(p => p.InternalNotes)
    .IsNotNavigable();     // can't be $expand'd
    .IsNotFilterable();    // can't be used in $filter
    .IsNotSortable();      // can't be used in $orderby
```

### Apply Authorization
```csharp
[Authorize]
[EnableQuery]
public IQueryable<Property> Get()
{
    // Scope data to authorized user
    var agentId = User.GetAgentId();
    return _context.Properties
        .Where(p => p.AgentId == agentId)   // row-level security in IQueryable
        .AsNoTracking();
}
```

### Never Expose Sensitive Fields
```csharp
// In EDM — exclude sensitive properties from OData model
builder.EntityType<Buyer>()
    .Ignore(b => b.CreditScore)
    .Ignore(b => b.SocialSecurityNumber)
    .Ignore(b => b.InternalNotes);
```

---

## OData Libraries Ecosystem

### .NET (Server)
| Library | Purpose | NuGet |
|---------|---------|-------|
| **Microsoft.AspNetCore.OData 8.x** | ASP.NET Core OData server (recommended for .NET 5+) | `Microsoft.AspNetCore.OData` |
| **Microsoft.OData.ModelBuilder** | Build EDM models from .NET classes | `Microsoft.OData.ModelBuilder` |
| **RESTier** | Higher-level OData bootstrap framework (on top of WebAPI OData) | `Microsoft.Restier.AspNet` |

### .NET (Client)
| Library | Purpose |
|---------|---------|
| **Microsoft.OData.Client** | LINQ-enabled OData client, auto-generated from metadata |
| **OData Connected Service** | Visual Studio extension — generates typed client from `$metadata` |
| **OData CLI** | CLI tool for generating client proxies |
| **Simple.OData.Client** | Multiplatform lightweight OData client |
| **OData.QueryBuilder** | Build OData query URLs with LINQ-like syntax |

### Cross-Platform / Other
| Library | Language |
|---------|---------|
| **odata.js** | JavaScript (browser + Node.js) |
| **odatapy** | Python |
| **DynamicODataToSQL** | Convert OData query → raw SQL (no EF needed) |

---

## When to Use OData vs GraphQL vs REST (Your Stack)

| Scenario | Use |
|---------|-----|
| Public property listing API (clients need filtering, sorting, paging) | **OData** or **GraphQL** |
| Internal reporting / analytics dashboard | **OData** (`$apply` aggregation) |
| Power BI / Excel data integration | **OData** (native support) |
| Flexible mobile/web client data fetching | **GraphQL** |
| Live auction bidding (real-time) | **GraphQL Subscriptions** |
| Simple CRUD admin API | **REST** |
| Enterprise B2B integration (SAP, Dynamics) | **OData** (standard) |
| Third-party client access with discovery | **OData** (`$metadata` self-description) |

### For Your Real Estate Domain
```
Public listing search API          → GraphQL (flexible client queries) or OData (enterprise clients)
Admin/reporting dashboard          → OData (analytics with $apply, Excel/Power BI integration)
Auction bidding (real-time)        → GraphQL Subscriptions
Mortgage rate lookup               → REST or OData
MLS/enterprise integrations        → OData (industry standard in real estate MLS systems!)
Property management internal tools → OData + EF Core (powerful data grid queries)
```

> **Note:** Real estate MLS (Multiple Listing Service) systems commonly use OData for data syndication. RESO (Real Estate Standards Organization) has standardized on OData for MLS data APIs — making OData a natural fit for your domain.

---

## Real Estate OData Examples

### Property Search
```
GET /odata/Properties
  ?$filter=City eq 'Austin' and Price ge 400000 and Price le 800000 and Bedrooms ge 3 and Status eq 'Active'
  &$select=Id,Address,Price,Bedrooms,Bathrooms,Status,MainPhotoUrl
  &$expand=Agent($select=Name,Phone)
  &$orderby=Price asc
  &$top=20
  &$count=true
```

### Auction Listings
```
GET /odata/Auctions
  ?$filter=Status eq 'Upcoming' and StartDate gt now()
  &$expand=Property($select=Address,MainPhotoUrl;$expand=Location),CurrentHighBid
  &$orderby=StartDate asc
  &$top=10
```

### Market Analytics (using $apply)
```
# Average sale price by city + property type
GET /odata/Properties
  ?$apply=filter(Status eq 'Sold')/groupby((City,PropertyType),aggregate(Price with average as AvgPrice,Price with max as MaxPrice,$count as SalesCount))
  &$orderby=SalesCount desc

# Monthly auction revenue
GET /odata/Auctions
  ?$apply=filter(Status eq 'Completed')/groupby((year(EndDate),month(EndDate)),aggregate(WinningBid with sum as Revenue))
  &$orderby=year(EndDate) desc,month(EndDate) desc
```

### Mortgage Rate Lookup
```
GET /odata/MortgageRates
  ?$filter=LoanType eq 'Fixed' and Term eq 30
  &$select=Lender,Rate,APR,Points,LastUpdated
  &$orderby=Rate asc
  &$top=5
```

---

## Key Rules (Never Forget)

1. **Return `IQueryable<T>`** from OData controllers — never `IEnumerable<T>` or `IList<T>`
2. **Apply `[EnableQuery]`** and configure limits — `MaxTop`, `MaxExpansionDepth`, `MaxNodeCount`
3. **Ignore sensitive fields** from the EDM model entirely
4. **Use `AsNoTracking()`** on all read queries
5. **ETag for concurrency** — always implement on mutable entities
6. **Server-driven paging** (`PageSize` in `[EnableQuery]`) — better than client-controlled `$skip` for large datasets
7. **`$metadata` endpoint** is your API's contract — keep the EDM clean and well-named
8. **RESO OData standard** — for real estate MLS data feeds, follow RESO Data Dictionary
9. **Never expose unlimited `$expand`** — set `MaxExpansionDepth = 2` or `3` maximum
10. **Test with OData `$metadata`** — import into Power BI or Excel to validate the model
