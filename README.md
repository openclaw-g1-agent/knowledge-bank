# 🧠 Knowledge Bank

A curated collection of technical knowledge bases used by the AI agent. Each file is a deep-dive reference compiled from official documentation and best practices.

---

## 📚 Contents

| File | Topics Covered |
|------|---------------|
| [dotnet10-csharp-api-knowledge.md](./dotnet10-csharp-api-knowledge.md) | .NET 10, C# 14, ASP.NET Core 10, Minimal APIs, CQRS, OpenAPI |
| [azure-services-knowledge.md](./azure-services-knowledge.md) | Azure App Configuration, Key Vault, Application Insights, AKS, Docker, CI/CD Pipelines, Workload Identity |
| [database-knowledge.md](./database-knowledge.md) | SQL Server 2025, MongoDB Atlas, Atlas Search, Vector Search, RAG pattern |
| [design-patterns-knowledge.md](./design-patterns-knowledge.md) | All 23 GoF Design Patterns with C# examples, SOLID principles |
| [orm-odm-knowledge.md](./orm-odm-knowledge.md) | EF Core 10 (SQL Server), MongoDB EF Core Provider, ORM vs ODM |
| [tech-frontend.md](./tech-frontend.md) | React 19, Next.js 15/16, ASP.NET Core MVC, Blazor |
| [tech-graphql.md](./tech-graphql.md) | GraphQL — type system, queries, mutations, subscriptions, security, performance, federation |
| [tech-odata-knowledge.md](./tech-odata-knowledge.md) | OData V4 — full query reference, ASP.NET Core implementation, real estate domain examples |

---

## 🏗️ Domain Context

These knowledge bases support a **real estate platform** consisting of:
- Public property listing websites
- Live auction platform
- Mortgage portal

### Tech Stack Decisions

| Layer | Choice |
|-------|--------|
| Backend API | ASP.NET Core 10 (C# 14) |
| API Layer | GraphQL (Hot Chocolate) |
| Frontend — Public/SEO | Next.js (SSG/ISR for listings, SSR for search) |
| Frontend — Internal | ASP.NET Core MVC / Blazor Server |
| Database — Relational | SQL Server 2025 + EF Core 10 |
| Database — Documents | MongoDB Atlas + Native C# Driver |
| Search | MongoDB Atlas Search |
| Semantic/AI Search | MongoDB Atlas Vector Search |
| Enterprise Integrations | OData V4 (MLS/Power BI/Excel) |
| Real-time (Auctions) | GraphQL Subscriptions |
| Cloud | Azure (AKS, App Config, Key Vault, App Insights) |
| UI Components (JS) | shadcn/ui + Tailwind CSS |
| UI Components (.NET) | MudBlazor / Radzen |

---

## 🗓️ Last Updated

- `2026-02-28` — .NET 10, Azure, Databases, Design Patterns, ORM/ODM
- `2026-03-01` — Frontend (React, Next.js, MVC, Blazor), GraphQL, OData
