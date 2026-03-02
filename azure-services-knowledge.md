# Azure Services Knowledge Base — Enterprise .NET 10 Applications
_Compiled: 2026-02-28_

---

## Architecture Overview: How All Services Connect

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE ENTERPRISE API STACK                    │
│                                                                   │
│  Dev Machine / CI Pipeline                                        │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────────┐  │
│  │  Source Code  │───▶│ Azure DevOps  │───▶│  Azure Container │  │
│  │  (GitHub/ADO) │    │  YAML Pipeline│    │  Registry (ACR)  │  │
│  └──────────────┘    └───────────────┘    └────────┬─────────┘  │
│                                                      │             │
│  Runtime                                             ▼             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                  Azure Kubernetes Service (AKS)             │ │
│  │  ┌────────────────────────────────────────────────────────┐ │ │
│  │  │ Pod: MyApi (.NET 10)                                    │ │ │
│  │  │  ├── Azure App Configuration (config + feature flags)  │ │ │
│  │  │  ├── Azure Key Vault (secrets via App Config refs)      │ │ │
│  │  │  ├── Application Insights / OpenTelemetry (telemetry)  │ │ │
│  │  │  └── Managed Identity (passwordless auth)              │ │ │
│  │  └────────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Azure App Configuration

### What It Does
Centralized key-value store for all application configuration. Supports:
- **Hierarchical keys**: `MyApp:Database:ConnectionString`
- **Labels**: per-environment configs (`dev`, `staging`, `prod`)
- **Feature flags**: toggle features without redeploy
- **Key Vault references**: secrets stay in KV, App Config holds the URI
- **Dynamic refresh**: apps reload config without restart (polling + middleware)

### NuGet Packages
```xml
<PackageReference Include="Microsoft.Azure.AppConfiguration.AspNetCore" Version="8.*" />
<PackageReference Include="Azure.Identity" Version="1.*" />
```

### Program.cs Integration (Enterprise Pattern)
```csharp
var builder = WebApplication.CreateBuilder(args);

// --- Azure App Configuration ---
string appConfigEndpoint = builder.Configuration["Endpoints:AppConfiguration"]
    ?? throw new InvalidOperationException("App Configuration endpoint not set.");

builder.Configuration.AddAzureAppConfiguration(options =>
{
    // Authenticate with Managed Identity (no secrets needed in code!)
    options.Connect(new Uri(appConfigEndpoint), new DefaultAzureCredential())

           // Load only keys for this app, specific environment label
           .Select("MyApp:*", LabelFilter.Null)           // shared config
           .Select("MyApp:*", builder.Environment.EnvironmentName) // env override

           // Key Vault integration — resolves KV references automatically
           .ConfigureKeyVault(kv =>
           {
               kv.SetCredential(new DefaultAzureCredential());
           })

           // Dynamic refresh: reload config when sentinel key changes
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("MyApp:Sentinel", refreshAll: true)
                      .SetRefreshInterval(TimeSpan.FromSeconds(30));
           })

           // Feature flags
           .UseFeatureFlags(flags =>
           {
               flags.CacheExpirationInterval = TimeSpan.FromMinutes(5);
           });
});

// Enable dynamic config middleware
builder.Services.AddAzureAppConfiguration();
builder.Services.AddFeatureManagement(); // for feature flags

// Bind config sections to strongly typed options
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("MyApp:Database"));
builder.Services.Configure<ServiceOptions>(
    builder.Configuration.GetSection("MyApp:Services"));

// ...

var app = builder.Build();

// MUST be early in the pipeline!
app.UseAzureAppConfiguration();
```

### Strongly Typed Options Pattern
```csharp
// Options classes (records preferred in .NET 10)
public record DatabaseOptions
{
    public string ConnectionString { get; init; } = string.Empty;
    public int CommandTimeout { get; init; } = 30;
    public int MaxRetryCount { get; init; } = 3;
}

public record ServiceOptions
{
    public string BaseUrl { get; init; } = string.Empty;
    public int TimeoutSeconds { get; init; } = 30;
}

// Inject with IOptionsSnapshot<T> for dynamic refresh (NOT IOptions<T>!)
public class UserService(IOptionsSnapshot<DatabaseOptions> dbOptions)
{
    // IOptionsSnapshot re-reads per request scope — reflects dynamic updates
    private readonly DatabaseOptions _db = dbOptions.Value;
}
```

### Feature Flags
```csharp
// In App Config portal: create a feature flag "BetaFeature"

// Usage in Minimal API endpoint
app.MapGet("/api/feature-test", async (IFeatureManager features) =>
{
    if (await features.IsEnabledAsync("BetaFeature"))
        return Results.Ok("Beta enabled!");

    return Results.Ok("Standard behavior.");
});

// Usage in service
public class MyService(IFeatureManager featureManager)
{
    public async Task<string> GetDataAsync()
    {
        return await featureManager.IsEnabledAsync("NewAlgorithm")
            ? await GetDataNewWayAsync()
            : await GetDataOldWayAsync();
    }
}
```

---

## 2. Azure Key Vault

### What It Does
- **Secrets**: passwords, connection strings, API keys, tokens
- **Keys**: encryption keys (HSM-protected in Premium tier)
- **Certificates**: TLS/SSL lifecycle management
- **FIPS 140-3 Level 3** HSMs in Premium tier — highest security
- Audit logging, access policies, RBAC
- **Never store secrets in code or config files**

### Two Auth Models
1. **Managed Identity** (preferred — no credentials at all)
2. **Service Principal** (for local dev / CI pipelines)

### Direct Key Vault Access (when not using App Config)
```xml
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.*" />
<PackageReference Include="Azure.Identity" Version="1.*" />
```

```csharp
// Program.cs — add KV as a config source directly
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVaultName"]}.vault.azure.net/"),
    new DefaultAzureCredential());
```

```csharp
// Or inject SecretClient for imperative access
builder.Services.AddSingleton(_ =>
    new SecretClient(
        new Uri($"https://my-vault.vault.azure.net/"),
        new DefaultAzureCredential()));

// In service
public class SecretsService(SecretClient secretClient)
{
    public async Task<string> GetSecretAsync(string name, CancellationToken ct = default)
    {
        KeyVaultSecret secret = await secretClient.GetSecretAsync(name, cancellationToken: ct);
        return secret.Value.Value;
    }
}
```

### App Config + Key Vault Integration (The Enterprise Pattern)
```
Azure App Configuration stores:
  Key: "MyApp:Database:Password"
  Value: (Key Vault reference) → https://myvault.vault.azure.net/secrets/DbPassword

Your app reads from App Config → provider auto-resolves KV reference → gets actual secret
```
```csharp
// App reads this as if it's a normal config value:
var password = builder.Configuration["MyApp:Database:Password"]; // auto-resolved from KV!
```

### RBAC Roles (Assign to Managed Identity)
| Role | Purpose |
|---|---|
| `Key Vault Secrets User` | Read secrets (most services need this) |
| `Key Vault Secrets Officer` | Read + write secrets |
| `Key Vault Administrator` | Full management |
| `App Configuration Data Reader` | Read App Config values |
| `App Configuration Data Owner` | Read + write App Config values |

### Local Development Auth (no managed identity locally)
```csharp
// DefaultAzureCredential tries in order:
// 1. EnvironmentCredential (env vars)
// 2. WorkloadIdentityCredential
// 3. ManagedIdentityCredential  ← used in Azure
// 4. VisualStudioCredential     ← used locally if signed into VS
// 5. AzureCliCredential         ← used locally if `az login` done
// 6. AzurePowerShellCredential
// 7. InteractiveBrowserCredential

// For local dev: run `az login` in terminal, then DefaultAzureCredential works automatically
var credential = new DefaultAzureCredential();
```

---

## 3. Application Insights + OpenTelemetry

### ⚠️ Important: Use OpenTelemetry Distro (NOT classic SDK)
Microsoft's current recommendation is to use `Azure.Monitor.OpenTelemetry.AspNetCore` for **all new .NET applications**. The classic `Microsoft.ApplicationInsights.*` SDK is still supported but deprecated for new projects.

### NuGet Package
```xml
<PackageReference Include="Azure.Monitor.OpenTelemetry.AspNetCore" Version="1.*" />
```

### Program.cs Setup (Minimal — One Line!)
```csharp
using Azure.Monitor.OpenTelemetry.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

// One line enables EVERYTHING: traces, metrics, logs, exceptions, dependencies
builder.Services.AddOpenTelemetry().UseAzureMonitor();

var app = builder.Build();
app.Run();
```

### Connection — Two Ways
```csharp
// Option 1: Environment variable (preferred for containers/AKS)
// Set: APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;IngestionEndpoint=...

// Option 2: In config / appsettings.json (never hardcode the connection string itself)
builder.Services.AddOpenTelemetry().UseAzureMonitor(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});
```

### What's Auto-Collected (Zero Code)
- HTTP requests (incoming)
- HTTP dependencies (outgoing `HttpClient` calls)
- SQL queries via EF Core / ADO.NET
- Exceptions (unhandled + caught)
- Performance counters (CPU, memory, GC)
- Heartbeats
- Live Metrics streaming

### Custom Telemetry
```csharp
using System.Diagnostics;

public class OrderService
{
    // Use ActivitySource (OTel standard) — NOT TelemetryClient
    private static readonly ActivitySource _activitySource = new("MyApp.Orders");

    public async Task<Order> ProcessOrderAsync(CreateOrderRequest request)
    {
        using var activity = _activitySource.StartActivity("ProcessOrder");
        activity?.SetTag("order.customerId", request.CustomerId);
        activity?.SetTag("order.itemCount", request.Items.Count);

        // ... business logic ...

        activity?.SetStatus(ActivityStatusCode.Ok);
        return order;
    }
}
```

### Custom Metrics (Instruments)
```csharp
using System.Diagnostics.Metrics;

public class OrderMetrics
{
    private static readonly Meter _meter = new("MyApp.Orders", "1.0.0");

    public static readonly Counter<long> OrdersCreated =
        _meter.CreateCounter<long>("orders.created", "orders", "Total orders created");

    public static readonly Histogram<double> OrderProcessingTime =
        _meter.CreateHistogram<double>("orders.processing_time_ms", "ms", "Order processing duration");

    public static readonly UpDownCounter<int> ActiveOrders =
        _meter.CreateUpDownCounter<int>("orders.active", "orders", "Currently active orders");
}

// Usage in service:
OrderMetrics.OrdersCreated.Add(1, new TagList { { "region", "us-east" } });
```

### Register Custom Instruments with App Insights
```csharp
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor()
    .WithMetrics(metrics =>
    {
        metrics.AddMeter("MyApp.Orders");
        metrics.AddMeter("MyApp.Users");
    })
    .WithTracing(tracing =>
    {
        tracing.AddSource("MyApp.Orders");
        tracing.AddSource("MyApp.Users");
        tracing.AddEntityFrameworkCoreInstrumentation();
    });
```

### Structured Logging → Application Insights
```csharp
// ILogger<T> automatically flows to Application Insights via OTel
public class UserController(ILogger<UserController> logger)
{
    public async Task<IResult> GetUser(Guid id)
    {
        // Structured log — searchable in App Insights by {UserId}
        logger.LogInformation("Fetching user {UserId}", id);

        var user = await _repo.GetByIdAsync(id);

        if (user is null)
        {
            logger.LogWarning("User {UserId} not found", id);
            return Results.NotFound();
        }

        return Results.Ok(user);
    }
}
```

### Key Application Insights Portal Features
| Feature | Use For |
|---|---|
| **Live Metrics** | Real-time request rate, failure rate, duration |
| **Application Map** | Visualize service dependencies |
| **Transaction Search** | Find specific requests / exceptions |
| **Performance** | Slowest operations, dependency bottlenecks |
| **Failures** | Exception details, failure trends |
| **Availability** | Uptime tests (ping from multiple regions) |
| **Workbooks** | Custom dashboards and reports |
| **Alerts** | Anomaly detection, threshold-based alerts |

---

## 4. Docker + Containerization

### Dockerfile (Enterprise Multi-Stage Build)
```dockerfile
# ── Stage 1: Restore (cache layer for fast rebuilds) ──
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS restore
WORKDIR /src
COPY ["MyApi.Api/MyApi.Api.csproj", "MyApi.Api/"]
COPY ["MyApi.Application/MyApi.Application.csproj", "MyApi.Application/"]
COPY ["MyApi.Domain/MyApi.Domain.csproj", "MyApi.Domain/"]
COPY ["MyApi.Infrastructure/MyApi.Infrastructure.csproj", "MyApi.Infrastructure/"]
RUN dotnet restore "MyApi.Api/MyApi.Api.csproj"

# ── Stage 2: Build ──
FROM restore AS build
COPY . .
WORKDIR /src/MyApi.Api
RUN dotnet build "MyApi.Api.csproj" -c Release --no-restore -o /app/build

# ── Stage 3: Publish ──
FROM build AS publish
RUN dotnet publish "MyApi.Api.csproj" \
    -c Release \
    --no-build \
    -o /app/publish \
    /p:UseAppHost=false

# ── Stage 4: Runtime (minimal image) ──
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app

# Security: run as non-root user
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser

COPY --from=publish /app/publish .

# Switch to non-root
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApi.Api.dll"]
```

### .dockerignore
```
**/.git
**/.vs
**/bin
**/obj
**/*.user
**/node_modules
**/.env
**/appsettings.Development.json
```

### Build & Push to ACR
```bash
# Login to ACR
az acr login --name myregistry

# Build and push (multi-arch for AKS)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myregistry.azurecr.io/myapi:$(git rev-parse --short HEAD) \
  -t myregistry.azurecr.io/myapi:latest \
  --push .
```

---

## 5. Kubernetes Manifests (AKS Deployment)

### Full Enterprise Deployment YAML
```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app: myapi
    environment: production
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  namespace: myapp
  labels:
    app: myapi
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 1 extra pod during update
      maxUnavailable: 0     # never take pods down during update
  template:
    metadata:
      labels:
        app: myapi
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: myapi-sa   # for Managed Identity / Workload Identity

      # Security context for the pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001

      containers:
        - name: myapi
          image: myregistry.azurecr.io/myapi:IMAGE_TAG   # replaced by pipeline
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              protocol: TCP

          # Security context for container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          # Environment variables — config from App Config, secrets from KV
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: Endpoints__AppConfiguration
              valueFrom:
                secretKeyRef:
                  name: myapi-secrets
                  key: app-config-endpoint
            - name: APPLICATIONINSIGHTS_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: myapi-secrets
                  key: appinsights-connection-string

          # Resource limits (ALWAYS set these in enterprise)
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Health probes
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          # Writable temp directories when using readOnlyRootFilesystem
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-logs
              mountPath: /app/logs

      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-logs
          emptyDir: {}

      # Graceful shutdown
      terminationGracePeriodSeconds: 60

      # Spread across nodes for HA
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapi
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapi-svc
  namespace: myapp
  labels:
    app: myapi
spec:
  type: ClusterIP    # Internal only — expose via Ingress
  selector:
    app: myapi
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
```

```yaml
# k8s/ingress.yaml — requires NGINX Ingress Controller or Azure AGIC
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.mycompany.com
      secretName: myapi-tls
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapi-svc
                port:
                  number: 80
```

```yaml
# k8s/hpa.yaml — Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```yaml
# k8s/pdb.yaml — Pod Disruption Budget (ensures HA during node maintenance)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapi-pdb
  namespace: myapp
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapi
```

---

## 6. Azure DevOps YAML Pipeline (Full CI/CD)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - docs/**
      - README.md

pr:
  branches:
    include:
      - main

variables:
  imageRepository: 'myapi'
  containerRegistry: 'myregistry.azurecr.io'
  dockerfilePath: 'src/MyApi.Api/Dockerfile'
  tag: '$(Build.BuildId)'
  vmImageName: 'ubuntu-latest'
  aksResourceGroup: 'myapp-rg'
  aksClusterName: 'myapp-aks'
  kubernetesNamespace: 'myapp'

stages:
  # ─────────── Stage 1: Build & Test ───────────
  - stage: CI
    displayName: Build, Test & Push
    jobs:
      - job: BuildAndTest
        displayName: Build, Test, and Publish Image
        pool:
          vmImage: $(vmImageName)
        steps:
          - task: UseDotNet@2
            displayName: Use .NET 10
            inputs:
              packageType: sdk
              version: '10.x'

          - task: DotNetCoreCLI@2
            displayName: Restore
            inputs:
              command: restore
              projects: '**/*.csproj'

          - task: DotNetCoreCLI@2
            displayName: Build
            inputs:
              command: build
              projects: '**/*.csproj'
              arguments: '--configuration Release --no-restore'

          - task: DotNetCoreCLI@2
            displayName: Run Unit Tests
            inputs:
              command: test
              projects: 'tests/**/*.UnitTests.csproj'
              arguments: >
                --configuration Release
                --no-build
                --collect:"XPlat Code Coverage"
                --results-directory $(Agent.TempDirectory)/coverage
              publishTestResults: true

          - task: DotNetCoreCLI@2
            displayName: Run Integration Tests
            inputs:
              command: test
              projects: 'tests/**/*.IntegrationTests.csproj'
              arguments: '--configuration Release --no-build'
              publishTestResults: true

          - task: PublishCodeCoverageResults@2
            displayName: Publish Code Coverage
            inputs:
              codeCoverageTool: Cobertura
              summaryFileLocation: $(Agent.TempDirectory)/coverage/**/coverage.cobertura.xml

          - task: Docker@2
            displayName: Build and Push Docker Image
            inputs:
              command: buildAndPush
              repository: $(imageRepository)
              dockerfile: $(dockerfilePath)
              containerRegistry: myDockerRegistryServiceConnection
              tags: |
                $(tag)
                $(Build.SourceBranchName)-latest

          - task: PublishPipelineArtifact@1
            displayName: Publish K8s Manifests
            inputs:
              artifactName: k8s-manifests
              path: k8s/

  # ─────────── Stage 2: Deploy to Dev ───────────
  - stage: DeployDev
    displayName: Deploy to Dev
    dependsOn: CI
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      environment: 'dev'
    jobs:
      - deployment: DeployToDev
        displayName: Deploy to AKS Dev
        pool:
          vmImage: $(vmImageName)
        environment: 'aks-dev.myapp'   # AKS environment in DevOps
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/deploy-aks.yml
                  parameters:
                    imageTag: $(tag)
                    environment: dev
                    aksResourceGroup: $(aksResourceGroup)
                    aksClusterName: '$(aksClusterName)-dev'
                    namespace: $(kubernetesNamespace)

  # ─────────── Stage 3: Deploy to Staging ───────────
  - stage: DeployStaging
    displayName: Deploy to Staging
    dependsOn: DeployDev
    condition: succeeded()
    jobs:
      - deployment: DeployToStaging
        displayName: Deploy to AKS Staging
        pool:
          vmImage: $(vmImageName)
        environment: 'aks-staging.myapp'
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/deploy-aks.yml
                  parameters:
                    imageTag: $(tag)
                    environment: staging
                    aksResourceGroup: $(aksResourceGroup)
                    aksClusterName: '$(aksClusterName)-staging'
                    namespace: $(kubernetesNamespace)

  # ─────────── Stage 4: Deploy to Production (approval gate) ───────────
  - stage: DeployProd
    displayName: Deploy to Production
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployToProd
        displayName: Deploy to AKS Production
        pool:
          vmImage: $(vmImageName)
        environment: 'aks-prod.myapp'   # Gate configured in DevOps portal
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/deploy-aks.yml
                  parameters:
                    imageTag: $(tag)
                    environment: production
                    aksResourceGroup: $(aksResourceGroup)
                    aksClusterName: '$(aksClusterName)-prod'
                    namespace: $(kubernetesNamespace)
```

### Reusable Deploy Template: `templates/deploy-aks.yml`
```yaml
parameters:
  - name: imageTag
    type: string
  - name: environment
    type: string
  - name: aksResourceGroup
    type: string
  - name: aksClusterName
    type: string
  - name: namespace
    type: string

steps:
  - download: current
    artifact: k8s-manifests

  - task: KubeloginInstaller@0
    displayName: Install kubelogin

  - task: AzureCLI@2
    displayName: Get AKS credentials
    inputs:
      azureSubscription: myAzureServiceConnection
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        az aks get-credentials \
          --resource-group ${{ parameters.aksResourceGroup }} \
          --name ${{ parameters.aksClusterName }} \
          --overwrite-existing
        kubelogin convert-kubeconfig -l azurecli

  - task: KubernetesManifest@1
    displayName: Deploy to AKS
    inputs:
      action: deploy
      namespace: ${{ parameters.namespace }}
      manifests: |
        $(Pipeline.Workspace)/k8s-manifests/deployment.yaml
        $(Pipeline.Workspace)/k8s-manifests/service.yaml
        $(Pipeline.Workspace)/k8s-manifests/ingress.yaml
        $(Pipeline.Workspace)/k8s-manifests/hpa.yaml
        $(Pipeline.Workspace)/k8s-manifests/pdb.yaml
      containers: |
        myregistry.azurecr.io/myapi:${{ parameters.imageTag }}
      # KubernetesManifest automatically replaces image tag in manifests

  - task: AzureCLI@2
    displayName: Verify Rollout
    inputs:
      azureSubscription: myAzureServiceConnection
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        kubectl rollout status deployment/myapi -n ${{ parameters.namespace }} --timeout=5m
        kubectl get pods -n ${{ parameters.namespace }}
```

---

## 7. Workload Identity (Managed Identity in AKS)

This is the modern, secure, passwordless way to access Azure services from AKS pods.

```bash
# Enable OIDC Issuer and Workload Identity on AKS cluster
az aks update \
  --resource-group myapp-rg \
  --name myapp-aks \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get OIDC issuer URL
OIDC_ISSUER=$(az aks show \
  --resource-group myapp-rg \
  --name myapp-aks \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create a User-Assigned Managed Identity
az identity create \
  --resource-group myapp-rg \
  --name myapi-identity

# Get the client ID
CLIENT_ID=$(az identity show \
  --resource-group myapp-rg \
  --name myapi-identity \
  --query clientId -o tsv)

# Assign RBAC roles to the identity
az role assignment create \
  --role "App Configuration Data Reader" \
  --assignee $CLIENT_ID \
  --scope /subscriptions/.../resourceGroups/myapp-rg/providers/Microsoft.AppConfiguration/configurationStores/myconfig

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $CLIENT_ID \
  --scope /subscriptions/.../resourceGroups/myapp-rg/providers/Microsoft.KeyVault/vaults/myvault

# Create Kubernetes Service Account with workload identity annotation
kubectl create serviceaccount myapi-sa -n myapp
kubectl annotate serviceaccount myapi-sa \
  azure.workload.identity/client-id=$CLIENT_ID -n myapp

# Create federated identity credential
az identity federated-credential create \
  --name myapi-federated \
  --identity-name myapi-identity \
  --resource-group myapp-rg \
  --issuer $OIDC_ISSUER \
  --subject system:serviceaccount:myapp:myapi-sa
```

```yaml
# Add to deployment pod spec:
spec:
  serviceAccountName: myapi-sa
  labels:
    azure.workload.identity/use: "true"  # enables workload identity
```

---

## 8. Full Program.cs — Enterprise Pattern

```csharp
using Azure.Identity;
using Azure.Monitor.OpenTelemetry.AspNetCore;
using Microsoft.Azure.AppConfiguration.AspNetCore;
using Microsoft.FeatureManagement;

var builder = WebApplication.CreateBuilder(args);

// ═══════════════════════════════════════════
// 1. AZURE APP CONFIGURATION + KEY VAULT
// ═══════════════════════════════════════════
string appConfigEndpoint = builder.Configuration["Endpoints:AppConfiguration"]
    ?? throw new InvalidOperationException("Endpoints:AppConfiguration not configured.");

var credential = new DefaultAzureCredential();

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(new Uri(appConfigEndpoint), credential)
           .Select("MyApp:*", LabelFilter.Null)
           .Select("MyApp:*", builder.Environment.EnvironmentName)
           .ConfigureKeyVault(kv => kv.SetCredential(credential))
           .ConfigureRefresh(refresh =>
               refresh.Register("MyApp:Sentinel", refreshAll: true)
                      .SetRefreshInterval(TimeSpan.FromSeconds(30)));
});

builder.Services.AddAzureAppConfiguration();
builder.Services.AddFeatureManagement();

// ═══════════════════════════════════════════
// 2. APPLICATION INSIGHTS (OpenTelemetry)
// ═══════════════════════════════════════════
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor()
    .WithTracing(tracing => tracing
        .AddSource("MyApp.*")
        .AddEntityFrameworkCoreInstrumentation())
    .WithMetrics(metrics => metrics
        .AddMeter("MyApp.*"));

// ═══════════════════════════════════════════
// 3. OPTIONS BINDING
// ═══════════════════════════════════════════
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("MyApp:Database"));

// ═══════════════════════════════════════════
// 4. APPLICATION SERVICES
// ═══════════════════════════════════════════
builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration)
    .AddApiServices();

builder.Services.AddOpenApi();
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database")
    .AddAzureAppConfiguration();

// ═══════════════════════════════════════════
// 5. PIPELINE
// ═══════════════════════════════════════════
var app = builder.Build();

app.UseAzureAppConfiguration();  // Must be early
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

if (app.Environment.IsDevelopment())
    app.MapOpenApi();

app.MapGroup("/api/v1")
   .MapUsersEndpoints()
   .RequireAuthorization();

app.MapHealthChecks("/health/live",  new() { Predicate = _ => false });
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
app.MapHealthChecks("/health",       new() { ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse });

app.Run();
```

---

_Sources: learn.microsoft.com/azure/azure-app-configuration, /azure/key-vault, /azure/azure-monitor, /azure/aks_
