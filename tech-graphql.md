# GraphQL — Full Deep Knowledge
_Learned: 2026-03-01 | Source: graphql.org/learn_

---

## What Is GraphQL?

- Query language for APIs + runtime for executing those queries
- Developed at Facebook (2012), open-sourced (2015), now governed by GraphQL Foundation
- **Single endpoint** (`/graphql`) — clients ask for exactly what they need
- Solves: over-fetching, under-fetching, multiple round-trips (all REST problems)
- Perfect for real estate: one query fetches property + agent + bids + mortgage options

### GraphQL vs REST
```
REST:   3 separate calls for property + bids + mortgage
GraphQL: 1 query, exact fields, all data in one response
```

---

## Three Operation Types

| Operation | Purpose | Analogy |
|-----------|---------|---------|
| `query` | Read data | GET |
| `mutation` | Write/modify data | POST/PUT/DELETE |
| `subscription` | Real-time streaming | WebSocket |

---

## Type System & Schema (SDL)

### Six Named Types

#### 1. Object Types
```graphql
type Property {
  id: ID!
  address: String!
  price: Float!
  status: PropertyStatus!
  agent: Agent
  bids: [Bid!]!
}
```
- `!` = Non-Null (never null, or GraphQL execution error)
- `[Bid!]!` = Non-Null list of Non-Null Bids

#### 2. Scalar Types
| Built-in | Description |
|---------|-------------|
| `Int` | 32-bit integer |
| `Float` | Double-precision float |
| `String` | UTF-8 |
| `Boolean` | true/false |
| `ID` | Unique identifier |

Custom: `scalar Date`, `scalar Money`, `scalar URL`, `scalar GeoCoordinates`

#### 3. Enum Types
```graphql
enum PropertyStatus { ACTIVE UNDER_CONTRACT SOLD AUCTION OFF_MARKET }
enum MortgageType { FIXED ADJUSTABLE FHA VA JUMBO }
```

#### 4. Interface Types
Abstract — concrete types must implement all fields:
```graphql
interface Listing {
  id: ID!
  title: String!
  price: Float!
}
type ResidentialProperty implements Listing { ... }
type CommercialProperty implements Listing { ... }
```

#### 5. Union Types
Field returns one of several types (no shared fields required):
```graphql
union SearchResult = Property | Agent | MortgageLender | AuctionEvent
```

#### 6. Input Object Types
Structured input for arguments (use `input` keyword, NOT `type`):
```graphql
input PropertySearchInput {
  city: String
  minPrice: Float
  maxPrice: Float
  bedrooms: Int
  radius: Float
}
input MortgageApplicationInput {
  propertyId: ID!
  loanAmount: Float!
  loanType: MortgageType!
}
```

### Root Operation Types
```graphql
type Query {
  property(id: ID!): Property
  properties(filter: PropertySearchInput, first: Int, after: String): PropertyConnection!
  mortgageRates(loanType: MortgageType): [MortgageRate!]!
}
type Mutation {
  placeBid(input: PlaceBidInput!): Bid!
  createMortgageApplication(input: MortgageApplicationInput!): MortgageApplication!
}
type Subscription {
  bidPlaced(auctionId: ID!): Bid!
  auctionStatusChanged(auctionId: ID!): Auction!
}
```

---

## Queries — Reading Data

### Key Features

#### Variables (always use — never string interpolate)
```graphql
query SearchProperties($filter: PropertySearchInput!, $first: Int) {
  properties(filter: $filter, first: $first) { address price status }
}
# Variables: { "filter": { "city": "Austin" }, "first": 10 }
```

#### Aliases (same field, different args)
```graphql
{ propertyA: property(id:"123") { price } propertyB: property(id:"456") { price } }
```

#### Fragments (reusable field sets)
```graphql
fragment PropertyCard on Property { id address price status mainImage }
query { featured: properties(filter:{featured:true}) { ...PropertyCard } }
```

#### Inline Fragments (for Interfaces/Unions)
```graphql
query { search(query:"Austin") {
  ... on Property { address price }
  ... on AuctionEvent { startDate reservePrice }
}}
```

#### Directives (dynamic query shape)
```graphql
query GetProperty($withAgent: Boolean!) {
  property(id:"123") {
    address
    agent @include(if: $withAgent) { name phone }
  }
}
```
- `@include(if: Boolean)` — include if true
- `@skip(if: Boolean)` — skip if true

#### `__typename` Meta Field
Essential for Union/Interface discrimination on client.

---

## Mutations — Writing Data

```graphql
mutation PlaceBid($input: PlaceBidInput!) {
  placeBid(input: $input) {
    id amount status
    auction { currentHighBid bidCount }
  }
}
```

- Top-level mutation fields run **serially** (not parallel) — no race conditions
- Use purpose-built mutations (e.g., `updateListingPrice`) not generic CRUD
- Always return the modified object (clients need the new state)
- No built-in transaction rollback — handle at business logic layer

---

## Subscriptions — Real-Time

```graphql
subscription LiveAuction($auctionId: ID!) {
  bidPlaced(auctionId: $auctionId) {
    amount
    bidder { name }
    auction { currentHighBid timeRemaining status }
  }
}
```

- Backed by pub/sub system (Redis, RabbitMQ, Kafka)
- Transport: WebSocket or Server-Sent Events
- Each subscription operation must have **exactly one root field**
- Scaling: clients bound to specific server instance → use pub/sub fan-out
- Use for: live auction bids, price changes, status updates
- Use polling instead for: infrequent updates (new listings, rate changes)

---

## Execution — How It Works

### Resolver Signature
```js
// (parent, args, context, info)
const resolvers = {
  Query: {
    property: (_, { id }, context) => context.db.properties.findById(id),
  },
  Property: {
    agent: (property, _, context) => context.db.agents.findById(property.agentId),
    bids: (property, { last }, context) => context.db.bids.forProperty(property.id, { last }),
  },
  Mutation: {
    placeBid: (_, { input }, context) => {
      context.requireAuth();
      return context.bidService.place(input);
    }
  }
}
```

| Arg | Description |
|-----|-------------|
| `parent` | Resolved value from parent field |
| `args` | Arguments from the query |
| `context` | Shared across all resolvers (auth, DB, services, DataLoaders) |
| `info` | Field metadata + schema info |

### Flow
1. Parse document
2. Validate against schema
3. Execute (queries: parallel; top-level mutations: serial)
4. Each resolver provides next value; recursion down to scalar leaves
5. Build JSON response mirroring query shape; send to client

### Trivial Resolvers
If no resolver defined, GraphQL reads same-named property from parent object automatically.

### Scalar Coercion
GraphQL converts resolver return values to expected schema types automatically.

---

## Pagination — Relay Connection Spec (Standard)

### Why Cursor-Based (not offset)
- Offset pagination breaks when records inserted/deleted between pages
- Cursors are stable opaque pointers (base64 encoded)

### Schema Pattern
```graphql
type PropertyConnection {
  edges: [PropertyEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}
type PropertyEdge {
  node: Property!
  cursor: String!
}
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### Usage
```graphql
query ListProperties($first: Int!, $after: String) {
  properties(first: $first, after: $after) {
    totalCount
    pageInfo { hasNextPage endCursor }
    edges { cursor node { id address price } }
  }
}
# Next page: pass endCursor as `after`
```

---

## Authorization

### Golden Rule: Logic in Business Layer — NOT resolvers
```js
// ❌ Wrong — duplicated, inconsistent
Property: { privateNotes: (p, _, ctx) => {
  if (p.agentId !== ctx.user.id) throw new Error('Forbidden');
  return p.privateNotes;
}}

// ✅ Correct — single source of truth
Property: { privateNotes: (p, _, ctx) => ctx.propertyService.getPrivateNotes(p.id, ctx.user) }
```

### Schema Directive Pattern
```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION
enum Role { ADMIN AGENT BUYER PUBLIC }
type Mutation {
  approveAuction(id: ID!): Auction! @auth(requires: ADMIN)
  placeBid(input: PlaceBidInput!): Bid! @auth(requires: BUYER)
}
```

### Context Object
Pass auth info through context to all resolvers:
```js
context: ({ req }) => ({ user: validateToken(req.headers.authorization), db, services })
```

---

## Security

| Threat | Defense |
|--------|---------|
| Deeply nested/cyclic queries | Depth limiting (max 5–7 levels) |
| Massive field selection | Breadth limiting + alias limits |
| Batching attacks | Batch operation count limit |
| Expensive resolvers | Query complexity analysis + cost budgets |
| Unlimited list data | Paginate ALL list fields |
| Schema enumeration | Disable introspection in production |
| Sensitive error leaking | Mask error details in production |
| Injection attacks | Sanitize inputs in business logic layer |
| First-party client APIs | Trusted documents (persisted query allowlist) |

### Trusted Documents (Persisted Queries)
```
Build time:  hash(query) → stored on server
Runtime:     client sends { documentId: "sha256abc", variables: {...} }
             server looks up, validates, executes
```
Best security posture for your own apps. Can't use for public APIs.

---

## Performance

### N+1 Problem & DataLoader (Critical!)
```
10 properties → 1 DB call
Each needs agent → 10 separate calls ← N+1 PROBLEM
```

Fix with DataLoader (batching + per-request caching):
```js
const agentLoader = new DataLoader(async (agentIds) => {
  const agents = await db.agents.findByIds(agentIds);
  return agentIds.map(id => agents.find(a => a.id === id));
});
// 10 property.agent calls → 1 batched DB query ✅
```

### HTTP GET for Queries
- GET requests are CDN-cacheable → use for public listing queries
- Use persisted queries (hashed) to avoid URL length limits

### Compression
- Always enable GZIP/Brotli — JSON compresses 60–80%
- Add `Accept-Encoding: gzip` header

### Monitoring
- Use **OpenTelemetry** for tracing resolver execution times
- Identify slow fields, N+1 patterns, error rates

---

## Introspection

GraphQL is self-documenting — query the schema itself:
```graphql
{ __schema { types { name fields { name type { name } } } } }
```
- Powers GraphiQL, Apollo Studio
- **Disable in production** for security (attackers can map your entire schema)

---

## Serving Over HTTP

### Request Format
```http
POST /graphql
Content-Type: application/json
{ "query": "...", "variables": {...}, "operationName": "GetProperty" }
```

### Response Format
```json
{
  "data": { "property": { "address": "123 Main St" } },
  "errors": null,
  "extensions": { "tracing": { "duration": 45 } }
}
```
- Always HTTP 200 (errors go in `errors` array, not HTTP status)

---

## Federation — Multi-Service Architecture

For multiple real estate apps, use Federation:
```
GraphQL Gateway (Router)
  ├── Property Subgraph (listings, search)
  ├── Auction Subgraph (bids, events, live status)
  └── Mortgage Subgraph (applications, rates, lenders)
```
Each service owns its schema slice. Gateway composes into one unified API.
Clients query one endpoint — unaware of service boundaries.

---

## Real Estate Domain Applications

### Public Property Search
```graphql
query SearchListings($filter: PropertySearchInput!, $first: Int!, $after: String) {
  properties(filter: $filter, first: $first, after: $after) {
    totalCount
    pageInfo { hasNextPage endCursor }
    edges { node { id address price status bedrooms photos(first:3) { url } agent { name } } }
  }
}
```

### Live Auction Bidding (Subscription)
```graphql
subscription LiveAuction($auctionId: ID!) {
  bidPlaced(auctionId: $auctionId) {
    amount bidder { name } placedAt
    auction { currentHighBid bidCount timeRemaining status }
  }
}
```

### Mortgage Pre-Qualification (Mutation)
```graphql
mutation ApplyForMortgage($input: MortgageApplicationInput!) {
  createMortgageApplication(input: $input) {
    id status prequalifiedAmount estimatedRate estimatedMonthlyPayment nextSteps
  }
}
```

---

## Tooling Recommendations

### Server (.NET) — Preferred for this project
| Tool | Purpose |
|------|---------|
| **Hot Chocolate** | Best GraphQL server for .NET (schema-first or code-first) |
| **Strawberry Shake** | GraphQL client for .NET |
| **GraphQL.NET** | Alternative .NET server |

### Server (Node.js)
| Tool | Purpose |
|------|---------|
| **Apollo Server** | Most popular |
| **GraphQL Yoga** | Lightweight, Envelop-powered |
| **Pothos** | Code-first TypeScript schema builder |

### Client
| Tool | Purpose |
|------|---------|
| **Apollo Client** | Full-featured (React, caching, subscriptions) |
| **TanStack Query + graphql-request** | Lighter alternative |
| **urql** | Minimal, extensible |
| **Relay** | Facebook's client (best for Relay Cursor Spec) |

### Developer Tools
| Tool | Purpose |
|------|---------|
| **GraphiQL** | Browser-based API explorer |
| **Apollo Studio** | Schema registry, monitoring, tracing |
| **GraphQL Code Generator** | Auto-generates TypeScript types from schema |

---

## Key Rules (Never Forget)

1. **Always use variables** — never string-interpolate query values
2. **Name all operations** — essential for debugging and logging
3. **Paginate all list fields** — security + performance
4. **Authorization in business logic layer** — not in resolvers
5. **Use DataLoader** — eliminates N+1 problem
6. **Disable introspection in production** — security
7. **Mutation fields run serially** — queries run in parallel
8. **One root field per subscription** operation
9. **Input types use `input` keyword** — not `type`
10. **Enable GZIP compression** — always in production
