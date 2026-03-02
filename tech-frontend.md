# Frontend Tech Stack Knowledge
_Learned: 2026-03-01 | Sources: react.dev, nextjs.org, learn.microsoft.com, dotnet.microsoft.com_

---

## ⚛️ React (v19+)

### What It Is
- JavaScript **UI library** (not a full framework) — handles the view layer only
- Component-based, declarative, maintained by Meta
- Powers Next.js, Remix, Expo (React Native) and more

### Core Concepts

#### Components
- JS functions that return JSX markup
- Must start with a capital letter (`<MyButton />`)
- Composable — nest inside each other
- Each instance has its own independent state

#### JSX
- HTML-like syntax compiled to JS
- Stricter than HTML — all tags must close
- One root element per return (use `<>...</>` fragments)
- `className` not `class`; curly braces `{}` for JS expressions

#### Props
- Data passed parent → child (read-only in child)
- Passed like HTML attributes: `<Card title="Hello" price={500000} />`

#### State (`useState`)
- Local mutable memory; triggers re-render on update
- `const [value, setValue] = useState(initialValue)`
- **Lifting state up** = move shared state to nearest common ancestor

#### Conditional Rendering
- Use JS `if/else`, ternary `? :`, or `&&` short-circuit inside JSX

#### List Rendering
- Use `.map()` with a unique `key` prop on each item

#### Events
- Pass function references: `onClick={handleClick}` (not `onClick={handleClick()}`)

### React Hooks (all start with `use`, called at top-level only)

| Hook | Purpose |
|------|---------|
| `useState` | Local component state |
| `useEffect` | Side effects (fetch, subscriptions, DOM) |
| `useContext` | Consume React Context (avoid prop drilling) |
| `useReducer` | Complex state logic |
| `useRef` | Mutable ref; DOM access; doesn't trigger re-render |
| `useMemo` | Memoize expensive computed values |
| `useCallback` | Memoize functions |
| `useLayoutEffect` | Like useEffect but synchronous after DOM update |
| `useId` | Generate unique IDs for accessibility |
| `useTransition` | Mark updates as non-urgent |
| `useDeferredValue` | Defer non-critical UI re-rendering |

### React 19 New Features
- **React Compiler** — auto-memoization at build time (replaces manual useMemo/useCallback)
- **Server Components** — render on server, zero JS sent to client
- **Actions** — simplified async form/mutation handling
- **`use()` hook** — read promises and context inside render

### Architecture Principles
- **Unidirectional data flow** — data down (props), events up
- **Declarative** — describe *what*, React handles *how*
- **Virtual DOM** — diffs virtual tree, applies minimal real DOM changes
- **Pure components** — same input → same output

---

## 🔺 Next.js (v15/16)

### What It Is
- Full-stack **React framework** by Vercel
- Adds routing, SSR, SSG, ISR, API routes, image optimization, etc.
- Use React for UI; Next.js for everything else

### Two Routers
- **App Router** (recommended) — `/app` dir, React Server Components, modern
- **Pages Router** (legacy, still supported) — `/pages` dir

### File-System Routing (App Router)
```
app/
├── layout.tsx        → Root layout (required, wraps everything)
├── page.tsx          → Route: /
├── about/page.tsx    → Route: /about
├── blog/
│   ├── page.tsx      → Route: /blog
│   └── [slug]/
│       └── page.tsx  → Route: /blog/:slug (dynamic)
```

### Special Files
| File | Purpose |
|------|---------|
| `page.tsx` | Route UI (makes route public) |
| `layout.tsx` | Shared UI; persists state across navigation |
| `loading.tsx` | Streaming Suspense loading UI |
| `error.tsx` | Error boundary |
| `not-found.tsx` | 404 page |
| `route.ts` | API endpoint |

### Rendering Strategies
| Strategy | When to Use | Real Estate Use |
|----------|-------------|-----------------|
| **SSG** | Pre-built at build time | Marketing/landing pages |
| **ISR** | Pre-built + background refresh | Property listings, prices |
| **SSR** | Per-request server render | Search results, personalized |
| **CSR** | Client-only (behind auth) | User dashboard, saved homes |
| **Streaming** | Progressive HTML delivery | Large pages, slow data |

### Server vs Client Components
```tsx
// Server Component (default) — runs on server only
// Can: fetch data, access DB, use secrets
// Cannot: use hooks, handle browser events
async function PropertyList() {
  const data = await db.properties.list(); // direct DB access OK
  return <ul>{data.map(p => <li>{p.address}</li>)}</ul>;
}

// Client Component — runs in browser
"use client"
function BidButton() {
  const [bid, setBid] = useState(0); // hooks OK
  return <button onClick={() => setBid(b => b + 1000)}>Raise Bid</button>;
}
```

### Data Fetching
```tsx
// ISR — revalidate every 60s (perfect for property listings)
const res = await fetch('https://api/properties', { next: { revalidate: 60 } });

// SSR — always fresh (search results)
const res = await fetch('https://api/search', { cache: 'no-store' });

// SSG — cache forever (static content)
const res = await fetch('https://api/static-content');
```

### Navigation
- `<Link href="/property/123">` — client-side + prefetch
- `useRouter()` — programmatic (Client Components)
- `redirect()` — server-side redirect

### Key Built-ins
| Feature | Description |
|---------|-------------|
| `next/image` | Optimized images (lazy, WebP, responsive) |
| `next/font` | Self-hosted fonts, zero layout shift |
| `next/link` | Client navigation + prefetch |
| Middleware | Auth, redirects, rewrites before request |
| Turbopack | Default bundler (faster than Webpack) |
| Server Actions | Async server functions callable from client forms |

### Setup
```bash
npx create-next-app@latest my-app --yes
# TypeScript + Tailwind + ESLint + App Router + Turbopack by default
```

### Best UI Pairings
- **shadcn/ui** — copy-paste, Tailwind-based, fully customizable
- **Radix UI** — headless accessible primitives
- **Framer Motion** — production animations
- **Tailwind CSS** — utility-first, rapid development

---

## 🏗️ ASP.NET Core MVC

### What It Is
- Server-side web framework for .NET
- Implements Model-View-Controller pattern
- Lightweight, open-source, testable
- Excellent for enterprise apps and REST APIs

### MVC Pattern
```
Request → Controller → Model (business logic) → Controller → View → Response
```

#### Model
- Application state + business logic
- Data annotations for validation:
```csharp
public class PropertyViewModel {
    [Required] public string Address { get; set; }
    [Range(0, double.MaxValue)] public decimal Price { get; set; }
}
```
- Independent of View and Controller (testable in isolation)

#### View (Razor Engine — `.cshtml`)
- HTML + embedded C# code
- Strongly typed views with IntelliSense
```cshtml
@model IEnumerable<Property>
@foreach (var p in Model) {
    <div class="property-card">@p.Address — $@p.Price</div>
}
```

#### Controller
- Handles user requests; selects view; works with model
- Should be thin — push logic to domain/service layer
```csharp
public async Task<IActionResult> Search(PropertySearchViewModel model) {
    if (!ModelState.IsValid) return View(model);
    var results = await _propertyService.Search(model);
    return View(results);
}
```

### Core Features

#### Routing
```csharp
// Convention-based
routes.MapRoute("Default", "{controller=Home}/{action=Index}/{id?}");

// Attribute routing (preferred for APIs)
[Route("api/properties")]
[HttpGet("{id}")]
public IActionResult GetProperty(int id) { ... }
```

#### Model Binding
Automatically maps request data (form, route, query string, headers) → C# objects.

#### Dependency Injection
```csharp
public class PropertiesController : Controller {
    private readonly IPropertyService _service;
    public PropertiesController(IPropertyService service) => _service = service;
}
```

#### Filters
Cross-cutting: auth, logging, caching, exception handling
```csharp
[Authorize(Roles = "Agent")]
public class ListingController : Controller { ... }
```

#### Tag Helpers
```cshtml
<a asp-controller="Property" asp-action="Detail" asp-route-id="@p.Id">View</a>
```

#### Areas
Partition large apps: `Admin`, `Listings`, `Auctions`, `Mortgage`

#### Web API Support
- JSON/XML content negotiation
- `[ApiController]` attribute
- CORS support

### Best For
- Enterprise line-of-business apps
- Admin dashboards
- Form-heavy data entry apps
- .NET teams not wanting JS framework overhead
- When paired with HTMX or Alpine.js for lightweight interactivity

---

## 🔥 Blazor (ASP.NET Core)

### What It Is
- .NET frontend framework — build UI with **C#** instead of JavaScript
- Part of ASP.NET Core
- Uses Razor components (`.razor` files)

### Hosting Models

#### Blazor Server
- C# runs on server; UI diffs sent via SignalR WebSocket
- ✅ Fast initial load, full .NET access, thin clients
- ❌ Requires persistent connection, latency on interactions

#### Blazor WebAssembly (WASM)
- .NET runtime + app downloaded to browser as WebAssembly
- Runs entirely in browser — true SPA
- ✅ Offline capable (PWA), no server round-trips after load
- ❌ Large initial download (mitigated by IL trimming + compression)

#### Blazor Web App (.NET 8+ — Recommended)
- Unified SSR + CSR model
- Mix render modes per-page or per-component:
  - `Static` — pure HTML
  - `Interactive Server` — SignalR
  - `Interactive WebAssembly` — WASM
  - `Interactive Auto` — starts Server, migrates to WASM

#### Blazor Hybrid
- Embeds in native .NET MAUI / WPF / WinForms apps
- No WebAssembly — uses embedded WebView
- For cross-platform desktop/mobile

### Component Example
```razor
<h1>Property Counter</h1>
<p>Bid count: @bidCount</p>
<button @onclick="PlaceBid">Place Bid</button>

@code {
    private int bidCount = 0;
    private void PlaceBid() => bidCount++;
}
```

### JavaScript Interop
```csharp
await JSRuntime.InvokeVoidAsync("alert", "Bid placed!");
```

### Best UI Libraries
- **MudBlazor** — Material Design component library for Blazor
- **Radzen Blazor** — Enterprise UI component suite
- **Blazorise** — Multi-framework Blazor components

### Best For
- .NET teams that don't want to write JavaScript
- Enterprise dashboards, internal tools
- Real-time apps (SignalR native integration)
- Desktop/mobile via Blazor Hybrid + .NET MAUI

---

## ⚖️ Decision Matrix

| Application Type | Primary Pick | Secondary | Notes |
|-----------------|-------------|-----------|-------|
| Public listing site | Next.js (SSG/ISR) | Astro | SEO critical |
| Auction platform | Next.js (SSR + Subscriptions) | — | Real-time bids |
| Mortgage portal | Next.js (SSR) | ASP.NET MVC | Personalized |
| Admin / backoffice | Blazor Server or MVC | Next.js | .NET preferred |
| Content/blog/docs | Astro | Next.js | Zero JS default |
| Internal tools | ASP.NET Core MVC | Blazor | Enterprise |
| Mobile + Web | Next.js + React Native | Blazor Hybrid | Shared logic |

## 🎨 Recommended Frontend Stack for Real Estate

- **Public websites** (listings, auctions, mortgage): **Next.js + React + shadcn/ui + Tailwind**
  - SSG/ISR for property listings
  - SSR for search results
  - Subscriptions for live auction bidding
- **Internal tools / admin**: **ASP.NET Core MVC** or **Blazor Server**
- **Backend API**: **ASP.NET Core Web API** with **GraphQL (Hot Chocolate)**
- **Content sites**: Consider **Astro**
