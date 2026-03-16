# Observability & Telemetry Standards

> **Mandatory team standard.** Every application — API and frontend — must implement observability exactly as described in this document.  
> Sources: [Azure Monitor Application Insights Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) · [OpenTelemetry Enable Guide](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=aspnetcore) · [JavaScript SDK](https://learn.microsoft.com/en-us/azure/azure-monitor/app/javascript-sdk) · [Grafana Azure Monitor Data Source](https://grafana.com/docs/grafana/latest/datasources/azure-monitor/)

---

## Table of Contents

1. [Why Observability Matters](#1-why-observability-matters)
2. [What We Use: Technology Choices](#2-what-we-use-technology-choices)
3. [Telemetry Data Types](#3-telemetry-data-types)
4. [Backend: ASP.NET Core API Setup](#4-backend-aspnet-core-api-setup)
5. [Frontend: Next.js Setup](#5-frontend-nextjs-setup)
6. [Custom Telemetry: Events, Metrics & User Sessions](#6-custom-telemetry-events-metrics--user-sessions)
7. [Distributed Tracing & Request Correlation](#7-distributed-tracing--request-correlation)
8. [GraphQL-Specific Telemetry](#8-graphql-specific-telemetry)
9. [Performance Monitoring Standards](#9-performance-monitoring-standards)
10. [Structured Logging Standards](#10-structured-logging-standards)
11. [Alerting Rules](#11-alerting-rules)
12. [Grafana Integration & Dashboards](#12-grafana-integration--dashboards)
13. [KQL Query Reference](#13-kql-query-reference)
14. [Configuration & Secrets Management](#14-configuration--secrets-management)
15. [What NOT To Do](#15-what-not-to-do)
16. [Implementation Checklist](#16-implementation-checklist)

---

## 1. Why Observability Matters

Observability is not optional. Without it:
- You find out about production failures from users, not systems
- You cannot diagnose slow API endpoints or database bottlenecks
- You have no data to make performance or capacity decisions
- You cannot prove feature usage or adoption to stakeholders

**The Three Pillars of Observability:**

| Pillar | What it answers | Tool |
|--------|----------------|------|
| **Logs** | What happened and when? | Application Insights → Log Analytics |
| **Metrics** | How much? How fast? How many? | Application Insights Metrics |
| **Traces** | Where did the request go, how long did each step take? | Application Insights Distributed Tracing |

All three must be implemented for every service.

---

## 2. What We Use: Technology Choices

| Layer | Technology | Reason |
|-------|-----------|--------|
| **APM Backend** | Azure Monitor Application Insights | Native Azure, deep .NET integration, OpenTelemetry-based |
| **Instrumentation standard** | OpenTelemetry (OTel) | Vendor-neutral, future-proof — Microsoft's recommended approach |
| **Backend SDK** | `Azure.Monitor.OpenTelemetry.AspNetCore` NuGet | Modern OTel distro for ASP.NET Core |
| **Frontend SDK** | `@microsoft/applicationinsights-web` npm package | React/Next.js compatible, tracks page views, sessions, clicks |
| **Dashboard / Reporting** | Grafana (future) + Azure Portal today | Azure Monitor data source plugin for Grafana |
| **Query Language** | KQL (Kusto Query Language) | Used for all log/trace queries in Grafana and Log Analytics |

> **Important:** We use the **Azure Monitor OpenTelemetry Distro** (not the legacy Classic API SDK). Microsoft has deprecated the Classic SDK. Do not use `Microsoft.ApplicationInsights` packages directly in new projects.

---

## 3. Telemetry Data Types

Application Insights collects and stores the following telemetry types. You must understand each:

| Type | Table in Log Analytics | What it captures |
|------|----------------------|-----------------|
| **Requests** | `requests` | Every inbound HTTP/GraphQL request — URL, method, duration, status |
| **Dependencies** | `dependencies` | Outbound calls — SQL, HTTP APIs, Redis, Service Bus |
| **Exceptions** | `exceptions` | All unhandled and caught exceptions with full stack trace |
| **Traces** | `traces` | Structured log messages (ILogger → Application Insights) |
| **Custom Events** | `customEvents` | Business events you emit manually (e.g., "PropertyViewed", "BidPlaced") |
| **Custom Metrics** | `customMetrics` | Numeric measurements you emit manually (e.g., active auction count) |
| **Page Views** | `pageViews` | Frontend page loads — URL, duration, user info |
| **Browser Timings** | `browserTimings` | Page load performance from real user browsers |
| **Availability** | `availabilityResults` | Synthetic health check ping results |

---

## 4. Backend: ASP.NET Core API Setup

### Step 1: Install the NuGet package

```bash
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore
```

For additional instrumentation (Entity Framework, SQL, Redis, etc.):

```bash
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Instrumentation.StackExchangeRedis
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
```

### Step 2: Configure in `Program.cs`

```csharp
using Azure.Monitor.OpenTelemetry.AspNetCore;
using OpenTelemetry.Resources;

var builder = WebApplication.CreateBuilder(args);

// ─── OBSERVABILITY SETUP ──────────────────────────────────────────────────────

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource =>
    {
        resource.AddService(
            serviceName: builder.Configuration["ApplicationInsights:ServiceName"] ?? "RealEstatePlatform",
            serviceVersion: builder.Configuration["ApplicationInsights:ServiceVersion"] ?? "1.0.0"
        );
    })
    .UseAzureMonitor(options =>
    {
        // Connection string is pulled from environment variable:
        // APPLICATIONINSIGHTS_CONNECTION_STRING
        // Do NOT hardcode the connection string here
        options.SamplingRatio = builder.Environment.IsProduction() ? 0.1f : 1.0f; // 10% sampling in prod, 100% in dev
    });

// Ensure ILogger integrates with Application Insights
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
});

// ─── APPLICATION SERVICES ────────────────────────────────────────────────────
builder.Services.AddControllers();
// ... rest of your registrations

var app = builder.Build();
app.Run();
```

### Step 3: Set the Cloud Role Name (mandatory for Application Map)

If you have more than one service sending telemetry to the same Application Insights resource (e.g., listings API + auctions API + mortgage API), you **must** set the Cloud Role Name so the Application Map shows them separately.

```csharp
// In Program.cs, inside ConfigureResource:
.ConfigureResource(resource =>
{
    resource.AddService(
        serviceName: "listings-api",   // Use: listings-api | auctions-api | mortgage-api
        serviceVersion: "2.1.0"
    );
    resource.AddAttributes(new Dictionary<string, object>
    {
        ["cloud.role"] = "listings-api",
        ["environment"] = builder.Environment.EnvironmentName.ToLower()
    });
})
```

### Step 4: Configure `appsettings.json`

```json
{
  "ApplicationInsights": {
    "ServiceName": "listings-api",
    "ServiceVersion": "2.1.0"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

> **Never put the connection string in `appsettings.json`.** Use environment variables or Azure Key Vault.

### Step 5: Environment variable (required in all environments)

```bash
# Local development: in launchSettings.json or .env
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;IngestionEndpoint=https://xxx.applicationinsights.azure.com/

# Azure App Service: set as Application Setting in Portal
# Azure Kubernetes: inject via Kubernetes Secret mounted as env var
```

### Step 6: Verify telemetry is flowing

After starting the app, hit a few endpoints. In Azure Portal:
- Go to your Application Insights resource
- Click **Live Metrics** — you should see real-time request data within seconds

---

## 5. Frontend: Next.js Setup

### Step 1: Install the npm package

```bash
npm install @microsoft/applicationinsights-web @microsoft/applicationinsights-react-js
```

### Step 2: Create the Application Insights provider

Create `src/lib/appInsights.ts`:

```typescript
import { ApplicationInsights } from '@microsoft/applicationinsights-web';
import { ReactPlugin } from '@microsoft/applicationinsights-react-js';

const reactPlugin = new ReactPlugin();

const appInsights = new ApplicationInsights({
  config: {
    connectionString: process.env.NEXT_PUBLIC_APPINSIGHTS_CONNECTION_STRING,
    extensions: [reactPlugin],
    extensionConfig: {},
    enableAutoRouteTracking: true,        // Track page views on Next.js route changes
    disableAjaxTracking: false,           // Track all fetch/XHR requests
    autoTrackPageVisitTime: true,         // Track how long users spend on pages
    enableCorsCorrelation: true,          // Correlate frontend requests with backend traces
    enableRequestHeaderTracking: true,
    enableResponseHeaderTracking: true,
    correlationHeaderExcludedDomains: ['*.queue.core.windows.net'],
    disableFetchTracking: false,          // Track fetch() calls (GraphQL queries go via fetch)
  },
});

export { reactPlugin, appInsights };
```

### Step 3: Initialize in `_app.tsx` or `layout.tsx`

**For Next.js App Router (`app/layout.tsx`):**

```tsx
'use client';

import { useEffect } from 'react';
import { appInsights } from '@/lib/appInsights';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    // Initialize only on client side
    if (typeof window !== 'undefined') {
      appInsights.loadAppInsights();
      appInsights.trackPageView(); // Track initial page view
    }
  }, []);

  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

**For Next.js Pages Router (`pages/_app.tsx`):**

```tsx
import type { AppProps } from 'next/app';
import { useEffect } from 'react';
import { AppInsightsContext } from '@microsoft/applicationinsights-react-js';
import { reactPlugin, appInsights } from '@/lib/appInsights';

export default function MyApp({ Component, pageProps }: AppProps) {
  useEffect(() => {
    if (typeof window !== 'undefined') {
      appInsights.loadAppInsights();
    }
  }, []);

  return (
    <AppInsightsContext.Provider value={reactPlugin}>
      <Component {...pageProps} />
    </AppInsightsContext.Provider>
  );
}
```

### Step 4: Track authenticated user sessions (mandatory)

Once a user logs in, immediately set their identity so all telemetry is linked to them:

```typescript
// Call this after successful login
import { appInsights } from '@/lib/appInsights';

export function setTelemetryUser(userId: string, accountId?: string) {
  appInsights.setAuthenticatedUserContext(
    userId,      // Must be a non-PII identifier (e.g., user GUID — NOT email or name)
    accountId,   // Optional: e.g., company/org ID
    true         // storeInCookie: persists across page loads
  );
}

// Call this on logout
export function clearTelemetryUser() {
  appInsights.clearAuthenticatedUserContext();
}
```

### Step 5: Environment variable in Next.js

```bash
# .env.local (never commit this file)
NEXT_PUBLIC_APPINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;IngestionEndpoint=https://...

# For production: set in Vercel / Azure Static Web Apps environment settings
```

> `NEXT_PUBLIC_` prefix is required for Next.js to expose the variable to the browser. The connection string is safe to expose client-side — it only allows telemetry **ingestion**, not data **retrieval**.

---

## 6. Custom Telemetry: Events, Metrics & User Sessions

### 6.1 Custom Events (Backend — C#)

Custom events track business-level actions. Use them to understand feature adoption and user behaviour.

```csharp
using Microsoft.ApplicationInsights;

public class PropertyService
{
    private readonly TelemetryClient _telemetry;

    public PropertyService(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }

    public async Task<Property> GetPropertyAsync(int propertyId, string userId)
    {
        // Track feature usage
        _telemetry.TrackEvent("PropertyViewed", new Dictionary<string, string>
        {
            ["propertyId"] = propertyId.ToString(),
            ["userId"] = userId,
            ["propertyType"] = "residential"
        });

        return await _repository.GetByIdAsync(propertyId);
    }
}
```

**Mandatory business events to track per domain:**

| Domain | Events to Track |
|--------|----------------|
| Listings | `PropertyViewed`, `PropertySearched`, `PropertySaved`, `PropertyShared` |
| Auctions | `AuctionViewed`, `BidPlaced`, `BidFailed`, `AuctionWon`, `AuctionLost` |
| Mortgage | `MortgageCalculated`, `MortgageApplicationStarted`, `MortgageApplicationSubmitted` |
| Auth | `UserRegistered`, `UserLoggedIn`, `UserLoggedOut`, `PasswordResetRequested` |

### 6.2 Custom Metrics (Backend — C#)

Track numeric measurements that aren't captured automatically.

```csharp
// Track a simple metric
_telemetry.TrackMetric("ActiveAuctions", activeCount);

// Track with properties for filtering in dashboards
_telemetry.TrackMetric(new MetricTelemetry
{
    Name = "BidAmount",
    Sum = bidAmount,
    Properties =
    {
        ["auctionId"] = auctionId.ToString(),
        ["currency"] = "USD"
    }
});
```

### 6.3 Custom Events (Frontend — TypeScript/Next.js)

```typescript
import { appInsights } from '@/lib/appInsights';

// Track a property card click
export function trackPropertyClick(propertyId: string, position: number, source: string) {
  appInsights.trackEvent({
    name: 'PropertyCardClicked',
    properties: {
      propertyId,
      position,           // Position in search results
      source,             // 'search_results' | 'featured' | 'map_view'
    }
  });
}

// Track search performed
export function trackSearch(query: string, filters: object, resultCount: number) {
  appInsights.trackEvent({
    name: 'PropertySearchPerformed',
    properties: {
      query,
      resultCount: resultCount.toString(),
      hasFilters: Object.keys(filters).length > 0 ? 'true' : 'false',
      ...filters
    }
  });
}

// Track bid placed
export function trackBidPlaced(auctionId: string, bidAmount: number) {
  appInsights.trackEvent({
    name: 'BidPlaced',
    properties: { auctionId },
    measurements: { bidAmount }
  });
}

// Track mortgage calculator used
export function trackMortgageCalculation(loanAmount: number, rate: number, term: number) {
  appInsights.trackEvent({
    name: 'MortgageCalculated',
    measurements: { loanAmount, rate, term }
  });
}
```

### 6.4 Custom Metrics (Frontend)

```typescript
// Track page performance manually
appInsights.trackMetric({
  name: 'PropertyListPageLoadTime',
  average: performance.now()
});

// Track search result rendering time
appInsights.trackMetric({
  name: 'SearchResultRenderTime',
  average: renderDurationMs,
  sampleCount: 1,
  properties: { resultCount: '24' }
});
```

### 6.5 Tracking Exceptions (Both Layers)

**Backend (C#) — handled exceptions:**

```csharp
try
{
    await _auctionService.PlaceBidAsync(bid);
}
catch (BidRejectionException ex)
{
    _telemetry.TrackException(ex, new Dictionary<string, string>
    {
        ["auctionId"] = bid.AuctionId.ToString(),
        ["userId"] = bid.UserId,
        ["reason"] = ex.RejectionReason
    });
    // Also log it
    _logger.LogWarning(ex, "Bid rejected for auction {AuctionId}", bid.AuctionId);
    throw;
}
```

**Frontend (TypeScript) — handled errors:**

```typescript
try {
  await placeBid(auctionId, amount);
} catch (error) {
  appInsights.trackException({
    exception: error as Error,
    properties: {
      auctionId,
      bidAmount: amount.toString(),
      source: 'AuctionBidForm'
    },
    severityLevel: 2 // SeverityLevel.Warning
  });
}
```

---

## 7. Distributed Tracing & Request Correlation

Distributed tracing lets you follow a single user request across your entire stack: Next.js browser → Next.js API route → ASP.NET Core GraphQL API → SQL database.

### How it works

Application Insights uses the **W3C Trace Context** standard. Each request carries a `traceparent` header that links all telemetry from that request into a single "operation". You can view the entire chain in **Application Insights → Transaction Search**.

### Frontend → Backend correlation (mandatory)

This is automatically handled when you set `enableCorsCorrelation: true` in the frontend SDK config (see Section 5, Step 2). The SDK adds the `traceparent` header to every outbound fetch request.

**Your backend must accept the CORS headers.** Add correlation headers to your CORS policy:

```csharp
// In Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins(
                "https://yourapp.com",
                "http://localhost:3000"
            )
            .AllowAnyMethod()
            .AllowAnyHeader()           // Required: allows traceparent header
            .AllowCredentials()
            .WithExposedHeaders(        // Required: expose correlation headers to JS
                "Request-Id",
                "Request-Context"
            );
    });
});
```

### Custom Spans / Activities (for detailed tracing in C#)

For complex operations (e.g., a bid placement that touches multiple services), create child spans:

```csharp
using System.Diagnostics;

private static readonly ActivitySource _activitySource = new("RealEstate.Auctions");

public async Task<BidResult> PlaceBidAsync(BidRequest bid)
{
    using var activity = _activitySource.StartActivity("PlaceBid");
    activity?.SetTag("auctionId", bid.AuctionId);
    activity?.SetTag("userId", bid.UserId);

    using var validationActivity = _activitySource.StartActivity("ValidateBid");
    var isValid = await _validator.ValidateAsync(bid);
    validationActivity?.SetTag("isValid", isValid);
    validationActivity?.Stop();

    using var dbActivity = _activitySource.StartActivity("PersistBid");
    var result = await _repository.SaveBidAsync(bid);
    dbActivity?.SetTag("bidId", result.BidId);
    dbActivity?.Stop();

    activity?.SetTag("success", result.Success);
    return result;
}
```

Register your ActivitySource in `Program.cs`:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("RealEstate.Auctions");
        tracing.AddSource("RealEstate.Listings");
        tracing.AddSource("RealEstate.Mortgage");
    });
```

---

## 8. GraphQL-Specific Telemetry

GraphQL does not use standard REST conventions (all requests are `POST /graphql`), so standard request tracking won't tell you which query/mutation was called. You **must** add GraphQL-specific telemetry.

### Middleware approach with Hot Chocolate

Create a custom diagnostic event listener:

```csharp
using HotChocolate.Diagnostics;
using Microsoft.ApplicationInsights;

public class AppInsightsGraphQLDiagnosticListener : ServerDiagnosticEventListener
{
    private readonly TelemetryClient _telemetry;

    public AppInsightsGraphQLDiagnosticListener(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }

    public override IDisposable ExecuteRequest(IRequestContext context)
    {
        return new RequestScope(_telemetry, context);
    }

    private class RequestScope : IDisposable
    {
        private readonly TelemetryClient _telemetry;
        private readonly IRequestContext _context;
        private readonly Stopwatch _stopwatch;
        private readonly DateTime _startTime;

        public RequestScope(TelemetryClient telemetry, IRequestContext context)
        {
            _telemetry = telemetry;
            _context = context;
            _stopwatch = Stopwatch.StartNew();
            _startTime = DateTime.UtcNow;
        }

        public void Dispose()
        {
            _stopwatch.Stop();
            var operationName = _context.Request.OperationName ?? "anonymous";
            var documentText = _context.Request.Query?.ToString() ?? string.Empty;

            // Determine if it's a query, mutation, or subscription
            var operationType = documentText.TrimStart().StartsWith("mutation")
                ? "mutation"
                : documentText.TrimStart().StartsWith("subscription")
                    ? "subscription"
                    : "query";

            // Track as a custom event for business analytics
            _telemetry.TrackEvent("GraphQLOperation", new Dictionary<string, string>
            {
                ["operationName"] = operationName,
                ["operationType"] = operationType,
                ["hasErrors"] = (_context.Result?.Errors?.Count > 0).ToString()
            }, new Dictionary<string, double>
            {
                ["durationMs"] = _stopwatch.ElapsedMilliseconds
            });

            // Track errors individually
            if (_context.Result?.Errors != null)
            {
                foreach (var error in _context.Result.Errors)
                {
                    _telemetry.TrackException(
                        error.Exception ?? new Exception(error.Message),
                        new Dictionary<string, string>
                        {
                            ["graphqlOperation"] = operationName,
                            ["graphqlError"] = error.Message,
                            ["graphqlCode"] = error.Code ?? "UNKNOWN"
                        }
                    );
                }
            }

            // Track performance as a dependency (shows in dependency charts)
            _telemetry.TrackDependency(
                dependencyTypeName: "GraphQL",
                target: "graphql",
                dependencyName: $"{operationType} {operationName}",
                data: operationName,
                startTime: DateTimeOffset.UtcNow.Subtract(_stopwatch.Elapsed),
                duration: _stopwatch.Elapsed,
                resultCode: _context.Result?.Errors?.Count > 0 ? "Error" : "Success",
                success: _context.Result?.Errors == null || _context.Result.Errors.Count == 0
            );
        }
    }
}
```

Register the listener:

```csharp
// Program.cs
builder.Services.AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddDiagnosticEventListener<AppInsightsGraphQLDiagnosticListener>();
```

---

## 9. Performance Monitoring Standards

### 9.1 Performance Budgets (SLA Targets)

These are your mandatory performance SLAs. Alert when breached (see Section 11):

| Endpoint Category | P95 Target | P99 Target |
|------------------|-----------|-----------|
| Property Search (GraphQL) | < 500ms | < 1,000ms |
| Property Detail Page | < 200ms | < 500ms |
| Live Auction Feed (WebSocket) | < 100ms latency | < 200ms latency |
| Mortgage Calculator API | < 100ms | < 200ms |
| Image Upload | < 3,000ms | < 5,000ms |
| Authentication (Login/Token) | < 300ms | < 500ms |

### 9.2 Backend Telemetry Initializer (Add Standard Properties to All Telemetry)

Create a telemetry initializer to enrich every piece of telemetry with standard context:

```csharp
using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.ApplicationInsights.Extensibility;

public class StandardTelemetryInitializer : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly IConfiguration _configuration;

    public StandardTelemetryInitializer(
        IHttpContextAccessor httpContextAccessor,
        IConfiguration configuration)
    {
        _httpContextAccessor = httpContextAccessor;
        _configuration = configuration;
    }

    public void Initialize(ITelemetry telemetry)
    {
        var context = _httpContextAccessor.HttpContext;

        // Always set environment
        telemetry.Context.GlobalProperties["Environment"] =
            _configuration["ASPNETCORE_ENVIRONMENT"] ?? "Unknown";

        // Always set service version
        telemetry.Context.Component.Version =
            _configuration["ApplicationInsights:ServiceVersion"] ?? "unknown";

        if (context == null) return;

        // Set authenticated user ID (non-PII: use claim type "sub" or user GUID)
        var userId = context.User?.FindFirst("sub")?.Value;
        if (!string.IsNullOrEmpty(userId))
        {
            telemetry.Context.User.AuthenticatedUserId = userId;
        }

        // Add tenant/account ID if multi-tenant
        var tenantId = context.User?.FindFirst("tenant_id")?.Value;
        if (!string.IsNullOrEmpty(tenantId))
        {
            telemetry.Context.User.AccountId = tenantId;
        }

        // Add request-specific properties
        if (telemetry is RequestTelemetry requestTelemetry)
        {
            requestTelemetry.Properties["UserAgent"] =
                context.Request.Headers["User-Agent"].ToString()[..Math.Min(200, context.Request.Headers["User-Agent"].ToString().Length)];
        }
    }
}
```

Register it:

```csharp
// Program.cs
builder.Services.AddSingleton<ITelemetryInitializer, StandardTelemetryInitializer>();
builder.Services.AddHttpContextAccessor();
```

### 9.3 Dependency Tracking (Auto-collected)

These are tracked **automatically** by the OpenTelemetry SDK — no code needed:

- SQL Server queries (via Entity Framework or ADO.NET)
- HTTP calls to external services
- Azure Service Bus messages
- Azure Blob Storage operations
- Redis cache calls (if `StackExchangeRedis` instrumentation is added)

**Ensure slow queries are visible** by enabling EF Core command logging:

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

---

## 10. Structured Logging Standards

Use `ILogger<T>` with **structured logging** (named parameters, not string interpolation). All log messages flow to Application Insights `traces` table automatically.

### ✅ Correct — structured logging

```csharp
_logger.LogInformation(
    "Property {PropertyId} viewed by user {UserId} from {IpAddress}",
    property.Id, userId, ipAddress);

_logger.LogWarning(
    "Bid rejected for auction {AuctionId}: {Reason}. User {UserId} attempted {BidAmount:C}",
    auctionId, rejectionReason, userId, bidAmount);

_logger.LogError(ex,
    "Failed to process mortgage application {ApplicationId} for user {UserId}",
    applicationId, userId);
```

### ❌ Wrong — string interpolation (loses structure, can't filter by property)

```csharp
_logger.LogInformation($"Property {property.Id} viewed by user {userId}"); // ❌
```

### Log Levels — When to Use Each

| Level | When to use | Cost in Application Insights |
|-------|-------------|------------------------------|
| `Trace` | Extremely verbose debugging | Very high — disable in production |
| `Debug` | Dev-time diagnostics | High — disable in production |
| `Information` | Normal business events, request handling | Medium — use selectively |
| `Warning` | Recoverable issues, retries, rejected business rules | Low — always enabled |
| `Error` | Failed operations that affect users | Very low — always enabled |
| `Critical` | System-wide failures, data corruption risk | Minimal — always enabled |

**Production log level policy:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "RealEstate": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  }
}
```

---

## 11. Alerting Rules

Create these alerts in Azure Monitor (Alerts → Create alert rule) on the Application Insights resource.

| Alert Name | Condition | Threshold | Severity | Action |
|-----------|-----------|-----------|---------|--------|
| High Error Rate | `requests` failure rate | > 5% over 5 min | Sev 1 | PagerDuty / Teams |
| Slow Response Time | `requests` P95 duration | > 2,000ms over 5 min | Sev 2 | Teams channel |
| Exception Spike | `exceptions` count | > 50 in 5 min | Sev 1 | PagerDuty |
| Dependency Failure | `dependencies` failure rate | > 10% over 5 min | Sev 2 | Teams channel |
| Availability Drop | `availabilityResults` success | < 95% over 5 min | Sev 1 | PagerDuty |
| High CPU / Memory | Server metrics | CPU > 80% for 10 min | Sev 2 | Teams channel |
| Zero Traffic | `requests` count | = 0 over 10 min (business hours) | Sev 1 | PagerDuty |

**KQL for custom alert — Bid Failure Spike:**
```kql
customEvents
| where name == "BidPlaced" and tostring(customDimensions.hasError) == "true"
| summarize count() by bin(timestamp, 5m)
| where count_ > 20
```

---

## 12. Grafana Integration & Dashboards

> Grafana integration is planned for a future phase. This section prepares your team for the setup.

### Step 1: Install the Azure Monitor Data Source Plugin in Grafana

The Azure Monitor data source plugin is built into Grafana (no install needed for Grafana 8+).

### Step 2: Configure authentication

In Grafana → Data Sources → Azure Monitor:

1. **Authentication method:** App Registration (Service Principal) — recommended for production
2. Create an Azure AD App Registration:
   - Go to Azure AD → App Registrations → New Registration
   - Name: `grafana-monitoring`
   - Grant it the **Monitoring Reader** role on the Application Insights resource
3. In Grafana, enter:
   - Tenant ID
   - Client ID (App Registration Application ID)
   - Client Secret

### Step 3: Query modes in Grafana

| Mode | Use case |
|------|---------|
| **Metrics** | Real-time numeric data (request rates, response times, CPU) |
| **Logs (KQL)** | Complex queries over requests, exceptions, custom events |
| **Traces** | Distributed trace visualization |
| **Azure Resource Graph** | Cross-subscription resource inventory |

### Step 4: Recommended Dashboard Panels

**Platform Overview Dashboard:**

| Panel | Type | KQL / Metric |
|-------|------|-------------|
| Request Rate (req/min) | Time series | `requests \| summarize count() by bin(timestamp, 1m)` |
| Error Rate (%) | Gauge | `requests \| summarize failed=countif(success==false), total=count()` |
| P95 Response Time | Time series | `requests \| summarize percentile(duration, 95) by bin(timestamp, 5m)` |
| Active Users | Stat | `pageViews \| summarize dcount(user_AuthenticatedId) by bin(timestamp, 1h)` |
| Top Slow Endpoints | Table | See KQL Section 13 |
| Exception Count | Time series | `exceptions \| summarize count() by bin(timestamp, 5m)` |
| Top GraphQL Operations | Bar chart | `customEvents \| where name == "GraphQLOperation"` |
| Dependency Health | Status grid | `dependencies \| summarize success=countif(success==true)` |

### Step 5: Grafana Dashboard Variables (for dynamic filtering)

```
Variable name: environment
Type: Custom
Values: development,staging,production

Variable name: service  
Type: Custom
Values: listings-api,auctions-api,mortgage-api

# Use in KQL:
| where customDimensions.Environment == '$environment'
| where cloud_RoleName == '$service'
```

---

## 13. KQL Query Reference

Essential queries for investigations and dashboard building. Run these in Log Analytics or Grafana Logs panels.

### Request Performance

```kql
// P50/P95/P99 response times per endpoint
requests
| where timestamp > ago(1h)
| summarize
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99),
    count_ = count()
  by name
| order by p95 desc
| take 20
```

```kql
// Failed requests with details
requests
| where success == false
| where timestamp > ago(24h)
| project timestamp, name, url, resultCode, duration, user_AuthenticatedId
| order by timestamp desc
```

### Exception Analysis

```kql
// Top exceptions in last 24h
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| order by count_ desc
| take 20
```

```kql
// Exception trend over time
exceptions
| where timestamp > ago(7d)
| summarize count() by bin(timestamp, 1h), type
| render timechart
```

### User Session Analysis

```kql
// Active users per hour
pageViews
| where timestamp > ago(24h)
| summarize uniqueUsers = dcount(user_AuthenticatedId) by bin(timestamp, 1h)
| render timechart
```

```kql
// Session duration distribution
sessions
| where timestamp > ago(7d)
| summarize sessionDuration = sum(duration) by session_Id, user_AuthenticatedId
| summarize
    avgSessionMin = avg(sessionDuration) / 60000,
    p95SessionMin = percentile(sessionDuration, 95) / 60000
```

### Feature Usage

```kql
// Property searches per day
customEvents
| where name == "PropertySearchPerformed"
| where timestamp > ago(30d)
| summarize count() by bin(timestamp, 1d)
| render timechart
```

```kql
// Bid success vs failure rate
customEvents
| where name == "BidPlaced"
| where timestamp > ago(7d)
| summarize
    total = count(),
    failed = countif(customDimensions.hasErrors == "true"),
    success = countif(customDimensions.hasErrors == "false")
| extend successRate = round(todouble(success) / total * 100, 1)
```

```kql
// GraphQL operation usage and performance
customEvents
| where name == "GraphQLOperation"
| where timestamp > ago(24h)
| summarize
    count_ = count(),
    avgDurationMs = avg(todouble(customMeasurements.durationMs)),
    errorCount = countif(customDimensions.hasErrors == "true")
  by tostring(customDimensions.operationName), tostring(customDimensions.operationType)
| order by count_ desc
```

### Dependency Health

```kql
// Slow database queries (over 500ms)
dependencies
| where type == "SQL" and duration > 500
| where timestamp > ago(1h)
| project timestamp, name, data, duration, success
| order by duration desc
| take 50
```

```kql
// Dependency failure rate per target
dependencies
| where timestamp > ago(1h)
| summarize
    total = count(),
    failures = countif(success == false),
    avgDurationMs = avg(duration)
  by target, type
| extend failureRate = round(todouble(failures) / total * 100, 1)
| order by failureRate desc
```

### Availability / Health

```kql
// Availability test results
availabilityResults
| where timestamp > ago(24h)
| summarize
    success = countif(success == 1),
    failure = countif(success == 0)
  by name, location
| extend availability = round(todouble(success) / (success + failure) * 100, 2)
```

---

## 14. Configuration & Secrets Management

### Local Development

Use `dotnet user-secrets` or a `.env.local` file — never commit secrets.

```bash
# Set Application Insights connection string as user secret
dotnet user-secrets set "APPLICATIONINSIGHTS_CONNECTION_STRING" "InstrumentationKey=xxx;..."
```

### Staging and Production

| Environment | Where to store the connection string |
|-------------|-------------------------------------|
| Azure App Service | Application Settings (encrypted at rest) |
| Azure Kubernetes | Kubernetes Secret → mounted as environment variable |
| Azure Container Apps | Secrets configuration in Container App |
| Local Dev | `dotnet user-secrets` or `.env.local` |

**Never store the connection string in:**
- `appsettings.json` (committed to git)
- `appsettings.Production.json` (committed to git)
- Dockerfile
- Docker Compose file in the repo root

### Multiple environments, single workspace vs. separate

Use **separate Application Insights resources** per environment:

| Environment | Resource |
|-------------|---------|
| Development | `appinsights-realestaste-dev` |
| Staging | `appinsights-realestate-staging` |
| Production | `appinsights-realestate-prod` |

Each has its own connection string. This prevents dev noise from polluting production dashboards and alerts.

---

## 15. What NOT To Do

❌ **Do NOT log PII (Personally Identifiable Information)**

```csharp
// ❌ Never log name, email, phone, SSN, address in telemetry
_logger.LogInformation("User {Email} logged in", user.Email);

// ✅ Use non-PII identifiers (GUIDs)
_logger.LogInformation("User {UserId} logged in", user.Id);
```

❌ **Do NOT log financial or sensitive data**

```csharp
// ❌ Never log credit card numbers, SSNs, bank accounts
_logger.LogInformation("Processing payment for card {CardNumber}", cardNumber);

// ✅ Log only safe identifiers
_logger.LogInformation("Processing payment {PaymentId}", paymentId);
```

❌ **Do NOT use string interpolation in log messages**

```csharp
_logger.LogError($"Failed for user {userId}"); // ❌ — loses structure, can't filter
_logger.LogError("Failed for user {UserId}", userId); // ✅
```

❌ **Do NOT track every single DB query as a custom event** — the SDK tracks dependencies automatically. Double-tracking wastes ingestion budget.

❌ **Do NOT hardcode connection strings** anywhere in code or config files.

❌ **Do NOT disable sampling in production** without understanding cost implications. Default sampling (10%) is fine for production.

❌ **Do NOT create different Application Insights resources per microservice** (unless budgets require it). One resource per environment is the recommended pattern; use `cloud_RoleName` to distinguish services.

---

## 16. Implementation Checklist

Use this checklist for every new service or major feature before marking it production-ready.

### Backend API Checklist

- [ ] `Azure.Monitor.OpenTelemetry.AspNetCore` NuGet package installed
- [ ] `AddOpenTelemetry().UseAzureMonitor()` configured in `Program.cs`
- [ ] `cloud_RoleName` / service name set (matches service name in Application Map)
- [ ] `APPLICATIONINSIGHTS_CONNECTION_STRING` set in environment (not in code/config files)
- [ ] `StandardTelemetryInitializer` registered and setting `AuthenticatedUserId`
- [ ] Structured logging used throughout (`{ParameterName}` not string interpolation)
- [ ] PII is not present in any log messages, events, or properties
- [ ] Mandatory business events tracked (see Section 6.1 table)
- [ ] GraphQL diagnostic listener registered (for GraphQL APIs)
- [ ] No Classic API SDK packages (`Microsoft.ApplicationInsights`) in project file
- [ ] CORS configured to expose `Request-Id` and `Request-Context` headers
- [ ] Production log level is `Warning` (not `Debug` or `Trace`)
- [ ] Availability test configured in Azure Portal for health check endpoint (`/healthz`)

### Frontend (Next.js) Checklist

- [ ] `@microsoft/applicationinsights-web` npm package installed
- [ ] `appInsights.ts` provider created with correct config (Section 5, Step 2)
- [ ] `appInsights.loadAppInsights()` called on client side in root layout
- [ ] `enableAutoRouteTracking: true` set (tracks page views on navigation)
- [ ] `enableCorsCorrelation: true` set (links frontend requests to backend traces)
- [ ] `NEXT_PUBLIC_APPINSIGHTS_CONNECTION_STRING` set in environment (not committed)
- [ ] `setAuthenticatedUserContext()` called after login (non-PII user ID)
- [ ] `clearAuthenticatedUserContext()` called after logout
- [ ] Mandatory frontend events tracked (property click, search, bid placed)
- [ ] Errors caught and tracked via `appInsights.trackException()`

### Grafana Readiness Checklist (Future)

- [ ] Azure AD App Registration created for Grafana with Monitoring Reader role
- [ ] Connection string safely stored in Grafana secrets (not in dashboard JSON)
- [ ] Cloud Role Name tags are consistent across all services
- [ ] Custom events and metrics follow agreed naming convention
- [ ] Dashboards use template variables for environment and service filtering

---

## Quick Reference: Naming Conventions

### Custom Event Names
- PascalCase nouns: `PropertyViewed`, `BidPlaced`, `MortgageCalculated`
- No spaces, no underscores, no hyphens
- Always use past tense (the event already happened)

### Custom Metric Names
- PascalCase: `ActiveAuctions`, `SearchResultCount`, `BidAmount`
- Include unit in name if not obvious: `PageLoadTimeMs`, `BidAmountUsd`

### Custom Property Names (in events/metrics)
- camelCase: `propertyId`, `userId`, `auctionId`, `bidAmount`
- All values as strings (Application Insights requirement for custom dimensions)

### Cloud Role Names (must be consistent across all services)
```
listings-api
auctions-api
mortgage-api
gateway-api
next-js-frontend
```

---

*Last updated: 2026-03-16 | Maintained by: Tech Lead*  
*Sources: [Azure Monitor Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) · [OpenTelemetry Enable Guide](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=aspnetcore) · [JavaScript SDK](https://learn.microsoft.com/en-us/azure/azure-monitor/app/javascript-sdk) · [Grafana Azure Monitor Data Source](https://grafana.com/docs/grafana/latest/datasources/azure-monitor/)*
