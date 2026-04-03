---
name: planner
description: Master execution plan for cross-repo ecosystem documentation. Enforces one agent per repo with maximum effort. Generates MCP-ready, Obsidian-linked, Elasticsearch-indexable docs with verified service graphs.
tools: Read, Write, Edit, Glob, Grep, Bash, Agent
---

# Ecosystem Documentation — Master Execution Plan

## Mission

Document 20 repositories with EXTREME technical granularity, verified cross-service links, Obsidian graph compatibility, and Elasticsearch MCP indexability. The output must be the single source of truth for an AI to understand the entire system — how every service works internally, how they connect, and what breaks when something changes.

---

## Core Principles

### 1. ONE AGENT PER REPO — NON-NEGOTIABLE

Every repository gets its own dedicated agent. Never batch repos. Never share agents between repos.

**Why:** Each agent must read the ENTIRE codebase of its repo with maximum effort and focus. Batching repos means shallow analysis. We need DEEP analysis — every aggregate, every endpoint, every event, every database table, every env var.

**How:** Spawn all agents in parallel within each phase. 20 repos = 20 simultaneous agents in Phase 1. 14 backend services = 14 simultaneous agents in Phase 2. 6 apps = 6 simultaneous agents in Phase 3.

Each agent receives:
- The full path to its repo
- The documentation schema (Section 3)
- The writing rules (Section 4)
- Instructions to read EVERYTHING — not sample, not skim, EVERYTHING

The agent MUST:
- Read every file in `domain/`, `application/`, `ports/`, `adapters/`, `cmd/`, `proto/`, `bootstrap/`, `pkg/`, `src/`, `app/` — whatever the project has
- Extract every struct, every interface, every function signature that matters
- Map every database table from GORM models, Prisma schemas, Room entities, or raw SQL
- List every gRPC RPC from proto files with full request/response message shapes
- List every REST route from Fiber/Express/Retrofit with payloads
- List every Socket.IO event with direction and payload
- List every EventBridge detail-type and SQS queue name
- List every env var from .env files, docker-compose, and config structs
- Trace the full request lifecycle from entry point (cmd/) through DI (bootstrap/) through handler (adapters/) through use case (application/) through domain logic (domain/)

### 2. VERIFIED CONNECTIONS ONLY

Every connection between services MUST have a code evidence trail: source file path + line number + the actual code snippet. No assumptions. No "likely connects to". No "probably calls".

If a connection cannot be verified from code, it gets `⚠️ UNVERIFIED: {what to check and who to ask}`.

Evidence sources (in priority order):
1. `.env`, `.env.example`, `.env.production` — `*_URL`, `*_HOST`, `*_PORT` variables
2. `docker-compose*.yml` — `depends_on`, environment vars, network aliases
3. `proto/**/*.proto` — imported packages, service definitions
4. `bootstrap/` or DI files — `grpc.Dial()`, `http.NewRequest()`, `sqs.NewClient()`
5. `adapters/**/` — queue names, topic names, EventBridge detail-types
6. Config structs — fields referencing external URLs
7. `go.mod` / `package.json` — internal module references

### 3. GRANULARITY = EXTREME, TOKENS = MINIMAL

This is the hard part. The documentation must be MORE detailed than reading the source code directly — but use FEWER tokens. This is achieved through:

- **Structured tables** instead of prose: a 30-line function description becomes a 5-row table
- **Mermaid diagrams** instead of text explanations: a state machine in 10 lines of Mermaid replaces 50 lines of prose
- **Field-level precision** instead of hand-wavy descriptions: "Order aggregate has `status` field of type `OrderStatus` with transitions: PENDING→CONFIRMED→PREPARING→READY→DISPATCHED→DELIVERED" is better than "The Order aggregate tracks order lifecycle"
- **Code references** instead of code copies: `(source: domain/order/aggregate.go:45)` points to the code, don't copy 50 lines of Go
- **Elimination of fluff**: no introductions, no conclusions, no "in this document we will explore", no transition phrases

### 4. DOCUMENTATION IS EXTENSIVE BUT ENXUTA

Extensive = covers EVERYTHING. Every aggregate field. Every RPC parameter. Every event payload field. Every database column. Every env var.

Enxuta = zero wasted tokens. Tables, not paragraphs. References, not copies. Structured metadata, not narrative.

The AI reading these docs should:
- Understand the full domain model without reading domain/ source
- Know every API endpoint without reading proto/ or route files
- Know every inter-service connection without reading bootstrap/ or config
- Understand the blast radius of any change without reading multiple repos
- BUT if it needs implementation details, the source references tell it exactly where to look

### 5. OBSIDIAN + ELASTICSEARCH NATIVE

Every document uses:
- `[[wikilinks]]` for cross-service references → creates Obsidian graph edges
- YAML frontmatter with `id`, `service`, `type`, `tags`, `links_to`, `linked_from` → Elasticsearch indexable
- Mermaid diagrams → renders in Obsidian natively
- Dataview-compatible frontmatter → queryable in Obsidian

---

## Ecosystem Inventory

### All 20 Repositories

| # | Repo | Stack | Type | Known Ports |
|---|------|-------|------|-------------|
| 1 | `anota-migrator` | Go 1.22, Cobra, pgx | CLI tool | — |
| 2 | `coupon-service-core` | Go 1.23, gRPC, GORM, EventBridge, SQS | gRPC microservice | gRPC |
| 3 | `customer-delivery-webpage` | React 18, TS, Vite, axios, Socket.IO, Leaflet, jotai | Web app | HTTP |
| 4 | `delivery-service-core` | — | Docs archive only | — |
| 5 | `intelligence-chat-socket` | Node.js (Bun), Express, Socket.IO, gRPC client, Redis | WebSocket gateway | 3100 |
| 6 | `intelligence-service-core` | Go 1.25, gRPC, MongoDB, Redis, Bedrock/OpenAI/Gemini/Ollama | gRPC microservice | 50059, 8080 |
| 7 | `kds-service-core` | Go 1.25, Fiber v2, GORM | HTTP microservice | 3333 |
| 8 | `logistics-rider-app` | Kotlin, Jetpack Compose, Retrofit, Room, Google Maps | Android app | — |
| 9 | `logistics-service-core` | Go 1.25, gRPC, PostgreSQL, DynamoDB, Redis, EventBridge, SQS | gRPC microservice | 50051 |
| 10 | `logistics-service-gateway` | Go 1.25, Fiber v3, gRPC client | REST→gRPC gateway | 3000 |
| 11 | `loyalty-service-core` | Go 1.23, gRPC, GORM, EventBridge, SQS, Lambda | gRPC microservice | gRPC |
| 12 | `mcai-pdv-service` | TS/React + Rust (Tauri) | Desktop app | — |
| 13 | `merchant-sso-webpage` | React/TS, Vite, Ory Client | Web app | HTTP |
| 14 | `meu-garcom-app` | Kotlin, Jetpack Compose, Retrofit, Room | Android app | — |
| 15 | `notify-service-core` | Go 1.24, gRPC, SQS, go-mail (SMTP) | Worker | — |
| 16 | `order-service-core` | Go 1.24, gRPC, GORM, MongoDB, Redis, EventBridge, SQS, DynamoDB | gRPC microservice | gRPC |
| 17 | `order-service-socket` | Node.js, Express, Socket.IO, Redis, Kratos | WebSocket gateway | 4000 |
| 18 | `payment-service-core` | Go 1.25, gRPC, Fiber, GORM, SQS, EventBridge, Matera | gRPC + webhook + saga | gRPC, HTTP |
| 19 | `poc-whatsapp-tauri` | TS/Rust/Node, Baileys | Desktop + bot | WS |
| 20 | `pos-service-core` | Go 1.26, Fiber v2, gRPC, GORM, OTel | HTTP microservice | 8080 |

### Verified Service Graph

```
logistics-rider-app ──REST──▶ logistics-service-gateway ──gRPC──▶ logistics-service-core
                                                                         │
                                                          EventBridge ◀──┘──▶ order-service-core
                                                          (order.status.changed)

intelligence-chat-socket ──gRPC──▶ intelligence-service-core ──gRPC──▶ customer-service-core (50051)

customer-delivery-webpage ──REST──▶ api.meucontrole.ai ──▶ {merchants, orders, customers, opendelivery}
customer-delivery-webpage ──Socket.IO──▶ api.meucontrole.ai (order-service-socket)

order-service-socket ──Redis Pub/Sub──▶ Redis ──Kratos──▶ Ory
merchant-sso-webpage ──REST──▶ Ory Kratos

kds-service-core ◀──HTTP──▶ pos-service-core (bidirectional webhooks)
pos-service-core ──HTTP──▶ Merchant API

meu-garcom-app ──REST──▶ pos-service-core

payment-service-core ──EventBridge/SQS──▶ (payment events)
coupon-service-core ──EventBridge/SQS──▶ (coupon events)
loyalty-service-core ──EventBridge/SQS──▶ (loyalty events)
notify-service-core ──SQS──▶ (notification queue) ──SMTP──▶ email
```

### Shared Infrastructure

| Component | Services |
|-----------|----------|
| PostgreSQL | anota-migrator, coupon, kds, logistics-core, loyalty, order-core, payment, pos |
| Redis | intelligence-core, intelligence-chat-socket, logistics-core, order-socket |
| MongoDB | intelligence-core, order-core |
| DynamoDB | logistics-core, order-core |
| AWS SQS | coupon, logistics-core, loyalty, notify, order-core, payment |
| AWS EventBridge | coupon, logistics-core, loyalty, order-core, payment |
| Ory Kratos | merchant-sso-webpage, order-service-socket |
| OpenTelemetry | intelligence-core, payment, pos |

---

## Documentation Schema

### YAML Frontmatter (MANDATORY on every .md)

```yaml
---
id: "{service-name}/{doc-type}"
service: "{service-name}"
type: "{overview|architecture|domain|apis|events|integrations|data|config|deploy|impact|screens|api-consumption}"
tags: [go, grpc, microservice, logistics]
links_to: ["order-service-core"]
linked_from: ["logistics-service-gateway"]
last_verified: "2026-04-02"
token_estimate: 1200
---
```

### Doc Types per Service Category

**Backend services (Go microservices, Node gateways, workers):**

| File | Content | When to Create |
|------|---------|----------------|
| `overview.md` | Identity, stack, type, ports, one-liner | ALWAYS |
| `architecture.md` | Layers (Mermaid), packages table, entry points, DI approach, patterns | ALWAYS |
| `domain.md` | Aggregates (ALL fields + types + invariants), state machines (Mermaid), VOs, domain events, domain errors | IF has domain layer |
| `apis.md` | Every gRPC RPC or REST route: method, path, ALL request fields, ALL response fields, auth, errors, grpcurl/curl examples | IF exposes APIs |
| `events.md` | Published events (name, detail-type, ALL payload fields, trigger). Consumed events (name, source, handler, side effects). Queue/topic names. Mermaid sequence diagrams | IF pub/sub |
| `integrations.md` | Every outbound connection: target, protocol, env var, data flow, failure behavior, evidence (file:line) | ALWAYS |
| `data.md` | ALL database tables with ALL columns + types + constraints. Key indexes. MongoDB collections with document shapes. DynamoDB table designs. Redis key patterns + TTLs. Connection config | IF uses databases |
| `config.md` | ALL env vars: name, type, default, required, description. Grouped by category | ALWAYS |
| `deploy.md` | Dockerfile layers, docker-compose services, ports, health check, volumes | IF has Dockerfile/compose |
| `impact.md` | Change impact matrix: what changes → affected service → how it breaks → evidence. Bidirectional | ALWAYS |

**Frontend/Mobile apps:**

| File | Content | When to Create |
|------|---------|----------------|
| `overview.md` | Identity, platform, stack, target users | ALWAYS |
| `architecture.md` | Component diagram (Mermaid), directory structure, state management, navigation, build system | ALWAYS |
| `screens.md` | ALL screens/pages: name, route, purpose, key components. Navigation flow (Mermaid) | ALWAYS |
| `api-consumption.md` | EVERY API call: endpoint, method, backend service, auth, request/response shapes, source file. `[[wikilinks]]` to backend `apis.md` | ALWAYS |
| `config.md` | Env vars, build config, API base URLs, feature flags | ALWAYS |
| `impact.md` | What backend changes break this app | ALWAYS |

### Token Budgets

| Doc File | Target | Max |
|----------|--------|-----|
| overview.md | 500 | 800 |
| architecture.md | 1000 | 1500 |
| domain.md | 1500 | 2500 |
| apis.md | 2000 | 3000 |
| events.md | 1000 | 1500 |
| integrations.md | 800 | 1200 |
| data.md | 1200 | 2000 |
| config.md | 600 | 1000 |
| deploy.md | 600 | 1000 |
| impact.md | 600 | 1000 |
| screens.md | 800 | 1500 |
| api-consumption.md | 1000 | 2000 |
| **Total per service** | **~8k** | **~12k** |

### Output Directory Structure

```
ecosystem-docs/
├── _meta/
│   ├── schema.md          # This schema (copy of this section)
│   ├── graph.md           # Full Mermaid topology + adjacency list
│   └── glossary.md        # Domain terms across all services
├── services/
│   └── {service-name}/    # One folder per backend (14 folders)
│       └── *.md           # All applicable doc files
├── apps/
│   └── {app-name}/        # One folder per frontend/mobile (6 folders)
│       └── *.md
├── tools/
│   └── {tool-name}/       # CLI tools, utilities (1 folder)
│       └── *.md
├── graphs/
│   ├── full-topology.md   # Complete Mermaid + adjacency list
│   ├── event-flows.md     # EventBridge/SQS choreography
│   ├── grpc-mesh.md       # gRPC call graph
│   ├── http-mesh.md       # REST/HTTP call graph
│   ├── data-dependencies.md  # Shared databases/caches
│   └── auth-flow.md       # Ory Kratos topology
└── index.json             # Elasticsearch bulk-index manifest
```

---

## Writing Rules (agents MUST follow)

| # | Rule | Rationale |
|---|------|-----------|
| 1 | **Tables for ALL structured data** | Endpoints, env vars, events, fields, columns — ALWAYS tables. A table with 20 rows uses fewer tokens than 20 sentences and is 10x easier for AI to parse |
| 2 | **Inline code for identifiers** | `CreateDelivery`, `order.status.changed`, `PORT=50051` — makes them searchable and unambiguous |
| 3 | **`[[wikilinks]]` for cross-service refs** | `See [[order-service-core/events]]` — creates Obsidian graph edges AND allows MCP to follow links |
| 4 | **No prose blocks > 3 lines** | If you wrote 4+ lines of paragraph, refactor into bullets or table. Prose is token-expensive and hard for AI to parse |
| 5 | **Code snippets ONLY when critical** | Proto definitions (abbreviated), state machine transitions, complex queries. Never copy boilerplate |
| 6 | **Source references for every claim** | `(source: adapters/grpc/handler.go:45)` — the AI can load the exact file if needed |
| 7 | **One concept per header** | `## Published Events` not `## Events, Messaging, and Queue Configuration` |
| 8 | **Impact is directional with evidence** | "If `PaymentStatus` enum changes → `order-service-core` saga consumer breaks (source: proto/payment/service.proto:23)" |
| 9 | **Mark unknowns explicitly** | `⚠️ UNVERIFIED: queue consumer not found in code — check with infra team` |
| 10 | **Zero duplication** | A fact lives in ONE file. Other files `[[wikilink]]` to it |
| 11 | **English everywhere** | All docs in English. Code examples as-is from source |
| 12 | **Field-level precision** | NOT "the Order has several status fields". YES: table with every field name, type, validation, description |
| 13 | **Mermaid for flows and states** | State machines, sequence diagrams, component diagrams — Mermaid is 5x more token-efficient than text descriptions |
| 14 | **Aggregate ALL fields** | Every aggregate/entity: list ALL fields with types, constraints, defaults. Not "main fields". ALL fields |
| 15 | **Aggregate ALL endpoints** | Every RPC/route: list ALL. Not "key endpoints". ALL endpoints |
| 16 | **Aggregate ALL events** | Every published/consumed event: list ALL with full payload schemas |
| 17 | **Aggregate ALL env vars** | Every env var: list ALL with type, default, required flag |
| 18 | **Aggregate ALL tables/collections** | Every database table: list ALL with ALL columns. Every MongoDB collection with ALL document fields |

---

## Execution Phases

### PHASE 0 — Scaffold

**Agents:** 1
**What:** Create entire `ecosystem-docs/` directory tree + `_meta/schema.md` with full schema from this plan.

---

### PHASE 1 — Connection Extraction

**Agents:** 20 (ONE PER REPO — ALL IN PARALLEL)
**What:** Each agent scans its repo for ALL inter-service connections.

Each agent MUST scan:
- `.env*` files for service URLs, queue names, host configs
- `docker-compose*.yml` for depends_on, environment vars, port mappings
- `proto/**/*.proto` for imported packages and services
- `bootstrap/` or DI/composition files for client instantiations (`grpc.Dial`, `http.Client`, `sqs.Client`)
- `adapters/**/` for queue names, EventBridge detail-types, bucket names
- Config structs for connection-related fields
- `go.mod` / `package.json` for internal org packages

Each agent outputs a `_connections.md` temporary file with structured findings:
```
CONNECTION:
  from: {repo}
  to: {target}
  protocol: {gRPC|REST|WebSocket|SQS|EventBridge|Redis|PostgreSQL|etc}
  evidence_file: {path}
  evidence_line: {line}
  evidence_snippet: {code}
  data_flow: {description}
  env_var: {if applicable}
  direction: {outbound|inbound|bidirectional}
```

**CRITICAL:** Do NOT guess connections. If the code doesn't explicitly reference another service, don't invent the link.

---

### PHASE 2 — Backend Service Documentation

**Agents:** 14 (ONE PER BACKEND SERVICE — ALL IN PARALLEL)
**What:** Each agent reads the ENTIRE codebase of its service and generates ALL applicable doc files.

**Agent effort level: MAXIMUM.** This is not a summary pass. Each agent must:

1. **Read EVERY file** in domain/, application/, ports/, adapters/, cmd/, proto/, bootstrap/, pkg/ — not a sample, not "key files", EVERY file
2. **Extract EVERY aggregate** with ALL fields, ALL types, ALL validation rules, ALL invariants
3. **Extract EVERY state machine** with ALL states and ALL transitions, rendered as Mermaid stateDiagram
4. **Extract EVERY value object** with ALL fields and ALL validation
5. **Extract EVERY domain event** with ALL payload fields
6. **Extract EVERY domain error** with code, message, trigger condition
7. **Extract EVERY gRPC RPC** from proto files with ALL request/response message fields (types, required/optional)
8. **Extract EVERY REST route** with method, path, ALL body fields, ALL query params, auth requirements
9. **Extract EVERY Socket.IO event** with direction, ALL payload fields
10. **Extract EVERY EventBridge event** with detail-type, ALL payload fields, trigger condition
11. **Extract EVERY SQS queue** name, consumer handler, message schema
12. **Extract EVERY database table** with ALL columns (name, type, nullable, default, constraints)
13. **Extract EVERY MongoDB collection** with document schema (from Go/JS structs)
14. **Extract EVERY DynamoDB table** with PK, SK, GSIs
15. **Extract EVERY Redis key pattern** with TTL, data type
16. **Extract EVERY env var** with type, default, required, category
17. **Map the FULL request lifecycle** from cmd/ entry → bootstrap/ DI → adapter handler → application use case → domain logic → outbound adapter
18. **Build the impact matrix** from verified connections: what changes here that could break other services, and what changes in other services that could break this one
19. **Use `[[wikilinks]]`** for every reference to another service
20. **Stay within token budgets** — use tables, Mermaid, and source references instead of prose

The agent MUST load `_connections.md` from Phase 1 to populate `integrations.md` and `impact.md` with verified evidence.

Backend services to document (14 agents):
1. `coupon-service-core`
2. `intelligence-chat-socket`
3. `intelligence-service-core`
4. `kds-service-core`
5. `logistics-service-core`
6. `logistics-service-gateway`
7. `loyalty-service-core`
8. `notify-service-core`
9. `order-service-core`
10. `order-service-socket`
11. `payment-service-core`
12. `pos-service-core`
13. `anota-migrator` (in tools/ — lighter docs)
14. `delivery-service-core` (docs archive — overview only)

---

### PHASE 3 — App/Frontend Documentation

**Agents:** 6 (ONE PER APP — ALL IN PARALLEL)
**What:** Each agent reads the ENTIRE app codebase and generates all applicable doc files.

**Agent effort level: MAXIMUM.** Each agent must:

1. **Map EVERY screen/page/composable** with route, purpose, components used
2. **Map EVERY API call** to its backend service with exact endpoint, method, auth, request/response shape, and source file reference
3. **Map the navigation graph** as Mermaid flowchart
4. **Extract ALL state management** patterns (jotai atoms, Redux slices, Room DAOs, DataStore preferences)
5. **Extract ALL env vars / build config** (VITE_*, BuildConfig, etc.)
6. **Build impact matrix**: which backend API changes would break which screens

Apps to document (6 agents):
1. `customer-delivery-webpage`
2. `logistics-rider-app`
3. `mcai-pdv-service`
4. `merchant-sso-webpage`
5. `meu-garcom-app`
6. `poc-whatsapp-tauri`

---

### PHASE 4 — Cross-Cutting Graphs

**Agents:** 7 (ALL IN PARALLEL)
**What:** Generate the 6 graph files + glossary by aggregating all docs from Phase 2+3.

| Agent | Output File | Input | Content |
|-------|-------------|-------|---------|
| `graph-topology` | `graphs/full-topology.md` + `_meta/graph.md` | All `integrations.md` + `impact.md` | Full Mermaid graph + adjacency list with evidence |
| `graph-events` | `graphs/event-flows.md` | All `events.md` | EventBridge/SQS sequence diagrams + event catalog |
| `graph-grpc` | `graphs/grpc-mesh.md` | All `apis.md` + `integrations.md` (gRPC only) | gRPC call graph + RPC table |
| `graph-http` | `graphs/http-mesh.md` | All `apis.md` + `api-consumption.md` (HTTP only) | REST call graph + endpoint table |
| `graph-data` | `graphs/data-dependencies.md` | All `data.md` | Shared DB/cache topology + risk analysis |
| `graph-auth` | `graphs/auth-flow.md` | All `config.md` + `integrations.md` (auth only) | Auth topology + sequence diagrams |
| `graph-glossary` | `_meta/glossary.md` | All `domain.md` + `overview.md` | Domain terms: term, definition, services that use it |

---

### PHASE 5 — Validation

**Agents:** 1 (sequential — needs full picture)
**What:** Verify ALL docs pass quality gates. Fix issues in-place.

Checks:
1. Every .md has valid YAML frontmatter with ALL required fields
2. Every `[[wikilink]]` resolves to an existing file
3. No doc file exceeds 3000 tokens
4. No prose block exceeds 3 lines
5. Every connection has `(source: file:line)` evidence
6. Bidirectional link consistency: A's `links_to` contains B → B's `linked_from` contains A
7. No empty files (frontmatter only)
8. No assumption words ("likely", "probably", "might") — replace with `⚠️ UNVERIFIED` or verify
9. All `_connections.md` temp files deleted

Output: `_meta/validation-report.md`

---

### PHASE 6 — Elasticsearch Index

**Agents:** 1 (sequential)
**What:** Generate `index.json` for Elasticsearch bulk indexing.

Each document becomes:
```json
{
  "id": "{frontmatter.id}",
  "service": "{frontmatter.service}",
  "type": "{frontmatter.type}",
  "tags": ["{frontmatter.tags}"],
  "links_to": ["{frontmatter.links_to}"],
  "linked_from": ["{frontmatter.linked_from}"],
  "token_estimate": "{frontmatter.token_estimate}",
  "file_path": "{relative path}",
  "title": "{first heading}",
  "content": "{markdown minus frontmatter}"
}
```

Also generates `index-mapping.json` with Elasticsearch field mappings.

---

## Execution Summary

| Phase | Agents | Parallel? | Depends On | What |
|-------|--------|-----------|------------|------|
| 0 | 1 | — | Nothing | Create directory scaffold + schema |
| 1 | **20** | **ALL PARALLEL** | Phase 0 | Extract verified connections per repo |
| 2 | **14** | **ALL PARALLEL** | Phase 0+1 | Deep-doc every backend service |
| 3 | **6** | **ALL PARALLEL** | Phase 0+1 | Deep-doc every frontend/mobile app |
| 4 | **7** | **ALL PARALLEL** | Phase 2+3 | Generate cross-cutting graph docs |
| 5 | 1 | Sequential | Phase 4 | Validate + fix all docs |
| 6 | 1 | Sequential | Phase 5 | Generate Elasticsearch index |
| **TOTAL** | **50** | | | **~150-200 doc files** |

```
Phase 0 (1)
    │
    ▼
Phase 1 (20 parallel)
    │
    ├──▶ Phase 2 (14 parallel) ──┐
    │                             ├──▶ Phase 4 (7 parallel) ──▶ Phase 5 (1) ──▶ Phase 6 (1)
    └──▶ Phase 3 (6 parallel)  ──┘
```

Phase 2 and 3 run simultaneously = **20 agents in parallel** at peak.

---

## MCP Integration

### Query Patterns

| AI Question | Docs Loaded | Tokens |
|-------------|-------------|--------|
| "What is X?" | `{X}/overview.md` | ~500 |
| "What API does X expose?" | `{X}/apis.md` | ~2000 |
| "What breaks if I change X?" | `{X}/impact.md` → follow links | ~2000 |
| "How do events flow?" | `graphs/event-flows.md` | ~1500 |
| "What uses PostgreSQL?" | `graphs/data-dependencies.md` | ~1000 |
| "Full topology" | `graphs/full-topology.md` | ~2000 |
| "X env vars?" | `{X}/config.md` | ~600 |

### Impact Tracing

```
traceImpact(service, change):
  1. Load {service}/impact.md
  2. Find rows matching {change}
  3. For each affected in results:
     Load {affected}/impact.md → recurse (max depth 3)
  4. Return full blast radius with evidence
```

### Token Efficiency

| Without Ecosystem Docs | With Ecosystem Docs |
|----------------------|---------------------|
| "What does logistics-core do?" → read source ~15k tokens | overview.md ~500 tokens |
| "What breaks if order status changes?" → read 5 repos ~100k+ | impact.md × 3 ~2500 tokens |
| Full system understanding → 20 repos ~500k+ | topology + overviews ~12k tokens |

---

## Obsidian Compatibility

### Wikilink Convention

`[[service-name/doc-type]]` → resolves to `services/service-name/doc-type.md`

### Tag Taxonomy

| Category | Tags |
|----------|------|
| Language | `#go`, `#typescript`, `#kotlin`, `#rust` |
| Protocol | `#grpc`, `#rest`, `#websocket`, `#socketio`, `#eventbridge`, `#sqs` |
| Infra | `#postgresql`, `#mongodb`, `#dynamodb`, `#redis`, `#s3` |
| Domain | `#logistics`, `#orders`, `#payments`, `#coupons`, `#loyalty`, `#pos`, `#intelligence`, `#notifications` |
| Type | `#microservice`, `#gateway`, `#worker`, `#webapp`, `#mobileapp`, `#desktop`, `#cli` |

---

## Infra Debt Register

Items that CANNOT be documented from code alone — flagged as `⚠️ INFRA DEBT`:

| Item | Missing | Where to Ask |
|------|---------|--------------|
| Kubernetes manifests | K8s YAML, Helm, replicas, limits | DevOps/SRE |
| CI/CD pipelines | GitHub Actions, deploy triggers | DevOps/SRE |
| DNS/Load Balancer | `api.meucontrole.ai` routing | Infra team |
| AWS Architecture | VPCs, subnets, security groups | Infra team |
| Monitoring/Alerting | CloudWatch, PagerDuty, dashboards | SRE |
| Secrets Management | SSM, Secrets Manager paths | Security team |
| Database hosting | RDS, ElastiCache, DocumentDB instances | DBA/Infra |

---

## Success Criteria

- [ ] 20/20 repos documented
- [ ] Every connection has source file + line evidence
- [ ] Zero assumed connections
- [ ] Every .md has valid YAML frontmatter
- [ ] Every `[[wikilink]]` resolves
- [ ] No doc exceeds 3000 tokens
- [ ] No prose block > 3 lines
- [ ] Obsidian graph renders full topology
- [ ] `index.json` is valid for Elasticsearch
- [ ] Impact tracing works end-to-end
- [ ] Total docs < 200k tokens
- [ ] Bidirectional links are consistent
- [ ] All temp files cleaned up
- [ ] Validation report shows zero critical issues
- [ ] ALL fields documented (not "key fields")
- [ ] ALL endpoints documented (not "main endpoints")
- [ ] ALL events documented (not "important events")
- [ ] ALL env vars documented (not "common env vars")
