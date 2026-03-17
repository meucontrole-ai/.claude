# CLAUDE.md

Guide for AI assistants working in this repository. Fullstack: Go (Hexagonal/DDD) + React (Vite/TypeScript).

## Language Rules

- **Code**: ALWAYS English — variable names, function names, types, constants, comments, commit messages, PR titles, branch names. NEVER Portuguese in code.
- **Communication**: ALWAYS reply to the user in **PT-BR** (Brazilian Portuguese). No exceptions.
- **Docs in `.claude/`**: English.
- **File names**: English.

## Auto-Routing (MANDATORY)

**BEFORE writing any code or answering any technical question**, you MUST:

1. Analyze the user's task
2. Consult the routing table below and identify ALL relevant docs
3. Read the identified docs using the Read tool
4. Only then execute the task following the documented patterns

This is automatic — the user does NOT need to ask. If the task involves aggregates, you read the aggregates doc. If it involves tests, you read the testing doc. No exceptions.

## Progressive Documentation Loading

Load ONLY the relevant documents. NEVER load everything at once.

Guides are organized by scope:
- **Backend** (Go): `.claude/docs/back/` — domain, adapters, ports, application, infra
- **Frontend** (React+Vite+TypeScript): `.claude/docs/front/` — components, state, data fetching, styling, routing

**Scope detection**: Determine scope from the task or files involved. If working in `domain/`, `ports/`, `adapters/`, `application/`, `bootstrap/`, `cmd/`, `pkg/` → backend. If working in `src/`, `components/`, `pages/`, `app/`, `hooks/`, `stores/` → frontend.

**ALWAYS load global rules for the detected scope**:
**IF** backend task **THEN** ALSO read `.claude/docs/back/00-INDEX.md` (naming, idioms, principles)
**IF** frontend task **THEN** ALSO read `.claude/docs/front/00-INDEX.md` (naming, TypeScript strict, principles)

**IF** creating aggregate or value object **THEN** read `.claude/docs/back/03-AGGREGATES.md` + `.claude/docs/back/04-VALUE-OBJECTS.md` (~6k tokens)
**IF** implementing use case **THEN** read `.claude/docs/back/06-COMMANDS-RESULTS.md` + `.claude/docs/back/00-WALKTHROUGH.md` (~5k tokens)
**IF** creating/modifying state machine **THEN** read `.claude/docs/back/07-STATE-MACHINE.md` (~3k tokens)
**IF** implementing domain events **THEN** read `.claude/docs/back/05-DOMAIN-EVENTS.md` (~3k tokens)
**IF** creating gRPC adapter **THEN** read `.claude/docs/back/08-GRPC-SERVICE.md` (~3k tokens)
**IF** implementing webhook **THEN** read `.claude/docs/back/09-WEBHOOK-SERVICE.md` + `.claude/docs/back/17-SECURITY.md` (~5k tokens)
**IF** implementing messaging/saga **THEN** read `.claude/docs/back/10-MESSAGING-SERVICE.md` (~3k tokens)
**IF** creating/modifying repository **THEN** read `.claude/docs/back/11-REPOSITORY-PATTERN.md` + `.claude/docs/back/12-OPTIMISTIC-LOCKING.md` (~5k tokens)
**IF** handling errors **THEN** read `.claude/docs/back/13-ERROR-HANDLING.md` (~3k tokens)
**IF** writing tests **THEN** read `.claude/docs/back/14-TESTING-PATTERNS.md` (~4k tokens)
**IF** adding logging **THEN** read `.claude/docs/back/15-LOGGING.md` (~3k tokens)
**IF** adding tracing/metrics **THEN** read `.claude/docs/back/16-TRACING-METRICS.md` (~4k tokens)
**IF** working with concurrency **THEN** read `.claude/docs/back/18-CONCURRENCY.md` (~3k tokens)
**IF** configuring infra/deploy **THEN** read `.claude/docs/back/19-INFRASTRUCTURE.md` (~4k tokens)
**IF** adding external dependency **THEN** read `.claude/docs/back/20-SDK-DEPENDENCIES.md` (~3k tokens)
**IF** generating package docs **THEN** read `.claude/docs/back/21-DOC-PACKAGE.md` (~2k tokens)
**IF** generating use case docs **THEN** read `.claude/docs/back/22-DOC-USECASE.md` (~2k tokens)
**IF** generating endpoint docs **THEN** read `.claude/docs/back/23-DOC-ENDPOINT.md` (~2k tokens)
**IF** generating aggregate docs **THEN** read `.claude/docs/back/24-DOC-AGGREGATE.md` (~2k tokens)
**IF** generating adapter docs **THEN** read `.claude/docs/back/25-DOC-ADAPTER.md` (~2k tokens)
**IF** reviewing PR **THEN** read `.claude/docs/back/26-PR-REVIEW.md` (~4k tokens)
**IF** need full flow understanding **THEN** read `.claude/docs/back/00-WALKTHROUGH.md` (~4k tokens)
**IF** need project structure understanding **THEN** read `.claude/docs/back/01-PROJECT-STRUCTURE.md` + `.claude/docs/back/02-COMPOSITION-ROOT.md` (~5k tokens)

### Frontend Routes

**IF** setting up frontend project structure **THEN** read `.claude/docs/front/01-PROJECT-ARCHITECTURE.md` (~4k tokens)
**IF** creating React component **THEN** read `.claude/docs/front/02-COMPONENTS.md` (~4k tokens)
**IF** managing state (local, shared, global, server) **THEN** read `.claude/docs/front/03-STATE-MANAGEMENT.md` (~4k tokens)
**IF** fetching data or configuring RTK Query **THEN** read `.claude/docs/front/04-DATA-FETCHING.md` (~4k tokens)
**IF** writing frontend tests **THEN** read `.claude/docs/front/05-TESTING.md` (~4k tokens)
**IF** implementing frontend security (auth, RBAC, XSS, CSRF) **THEN** read `.claude/docs/front/06-SECURITY.md` (~3k tokens)
**IF** adding frontend observability (Sentry, analytics, Web Vitals) **THEN** read `.claude/docs/front/07-OBSERVABILITY.md` (~3k tokens)
**IF** configuring frontend build/deploy (Docker, CI/CD, Nginx) **THEN** read `.claude/docs/front/08-BUILD-DEPLOY.md` (~3k tokens)
**IF** adding frontend dependency **THEN** read `.claude/docs/front/09-DEPENDENCIES.md` (~3k tokens)
**IF** optimizing frontend performance (lazy loading, memoization, virtualization) **THEN** read `.claude/docs/front/10-PERFORMANCE.md` (~3k tokens)
**IF** handling frontend errors (Error Boundary, AppError, toasts) **THEN** read `.claude/docs/front/11-ERROR-HANDLING.md` (~3k tokens)
**IF** styling with Tailwind/CVA/theming **THEN** read `.claude/docs/front/12-STYLING.md` (~3k tokens)
**IF** implementing accessibility (WCAG, ARIA, keyboard nav) **THEN** read `.claude/docs/front/13-ACCESSIBILITY.md` (~3k tokens)
**IF** configuring routing (React Router, guards, lazy routes) **THEN** read `.claude/docs/front/14-ROUTING.md` (~3k tokens)

### Reference Tables

#### Backend (Go + Hexagonal + DDD)

| Task | Primary Document | Cost |
| --- | --- | --- |
| Project structure | .claude/docs/back/01-PROJECT-STRUCTURE.md | ~3k |
| Composition root / DI | .claude/docs/back/02-COMPOSITION-ROOT.md | ~3k |
| Aggregates | .claude/docs/back/03-AGGREGATES.md | ~3k |
| Value Objects | .claude/docs/back/04-VALUE-OBJECTS.md | ~3k |
| Domain Events | .claude/docs/back/05-DOMAIN-EVENTS.md | ~3k |
| Commands & Results | .claude/docs/back/06-COMMANDS-RESULTS.md | ~3k |
| State Machine | .claude/docs/back/07-STATE-MACHINE.md | ~3k |
| gRPC Service | .claude/docs/back/08-GRPC-SERVICE.md | ~3k |
| Webhook Service | .claude/docs/back/09-WEBHOOK-SERVICE.md | ~3k |
| Messaging / Saga | .claude/docs/back/10-MESSAGING-SERVICE.md | ~3k |
| Repository | .claude/docs/back/11-REPOSITORY-PATTERN.md | ~3k |
| Optimistic Locking | .claude/docs/back/12-OPTIMISTIC-LOCKING.md | ~2k |
| Error Handling | .claude/docs/back/13-ERROR-HANDLING.md | ~3k |
| Testing | .claude/docs/back/14-TESTING-PATTERNS.md | ~4k |
| Logging | .claude/docs/back/15-LOGGING.md | ~3k |
| Tracing & Metrics | .claude/docs/back/16-TRACING-METRICS.md | ~4k |
| Security | .claude/docs/back/17-SECURITY.md | ~3k |
| Concurrency | .claude/docs/back/18-CONCURRENCY.md | ~3k |
| Infrastructure | .claude/docs/back/19-INFRASTRUCTURE.md | ~4k |
| SDK & Dependencies | .claude/docs/back/20-SDK-DEPENDENCIES.md | ~3k |
| Doc: Package | .claude/docs/back/21-DOC-PACKAGE.md | ~2k |
| Doc: Use Case | .claude/docs/back/22-DOC-USECASE.md | ~2k |
| Doc: Endpoint | .claude/docs/back/23-DOC-ENDPOINT.md | ~2k |
| Doc: Aggregate | .claude/docs/back/24-DOC-AGGREGATE.md | ~2k |
| Doc: Adapter | .claude/docs/back/25-DOC-ADAPTER.md | ~2k |
| PR Review | .claude/docs/back/26-PR-REVIEW.md | ~4k |

#### Frontend (React + Vite + TypeScript)

| Task | Primary Document | Cost |
| --- | --- | --- |
| Project architecture | .claude/docs/front/01-PROJECT-ARCHITECTURE.md | ~4k |
| Components | .claude/docs/front/02-COMPONENTS.md | ~4k |
| State management | .claude/docs/front/03-STATE-MANAGEMENT.md | ~4k |
| Data fetching / RTK Query | .claude/docs/front/04-DATA-FETCHING.md | ~4k |
| Testing (Vitest, Playwright) | .claude/docs/front/05-TESTING.md | ~4k |
| Security (auth, RBAC, XSS) | .claude/docs/front/06-SECURITY.md | ~3k |
| Observability (Sentry, vitals) | .claude/docs/front/07-OBSERVABILITY.md | ~3k |
| Build & Deploy | .claude/docs/front/08-BUILD-DEPLOY.md | ~3k |
| Dependencies | .claude/docs/front/09-DEPENDENCIES.md | ~3k |
| Performance | .claude/docs/front/10-PERFORMANCE.md | ~3k |
| Error handling | .claude/docs/front/11-ERROR-HANDLING.md | ~3k |
| Styling (Tailwind, CVA) | .claude/docs/front/12-STYLING.md | ~3k |
| Accessibility (WCAG) | .claude/docs/front/13-ACCESSIBILITY.md | ~3k |
| Routing (React Router) | .claude/docs/front/14-ROUTING.md | ~3k |

**Token efficiency**: Load only what you need. Typical task = 5-10k tokens instead of ~80k+.

---

## Repository Overview

Payment service in Go with hexagonal architecture (ports & adapters) and DDD.

### Services

- **Core** (gRPC): Main payment API
- **Webhook** (HTTP/Fiber): Receives provider notifications
- **Saga** (SQS Consumer): Async validations

### Structure

- `domain/` — Aggregates, value objects, events, errors (pure rules, zero external dependency)
- `ports/` — Interfaces only (inbound: use cases, outbound: repos/providers)
- `application/` — Use cases grouped by entity (delivery/, driver/, merchant/, driver_merchant/)
- `adapters/` — gRPC handlers, repositories, providers, consumers
- `bootstrap/` — Manual DI (no framework)
- `cmd/` — Binaries (service_core, service_webhook, service_saga)
- `pkg/` — Shared utilities (telemetry, logger)

---

## Commands

```bash
# Build
go build ./...

# Test (ALWAYS with -race)
go test ./... -race

# Lint
golangci-lint run

# Security
gosec ./...
govulncheck ./...

# Proto
make proto
```

---

## Important Notes

- Dependency direction MANDATORY: adapters -> ports -> domain (NEVER the reverse)
- ALWAYS DomainError for errors — NEVER return raw infra errors
- ALWAYS fail-fast in aggregate constructor (validate everything before creating)
- NEVER change status directly — use mutation methods (business verbs)
- NEVER log PII (PAN, CVV, CPF, tokens, secrets)
- NEVER store PAN/CVV — use provider tokens
- NEVER use `==` for HMAC comparison — use `hmac.Equal()` (constant-time)
- ALWAYS `WHERE id = ? AND version = ?` for updates (optimistic locking)
- Tests in same package (not separate `_test`), manual spies (not generated mocks)
- `-race` flag mandatory in tests
- NEVER commit .env files or secrets

---

## Workflow Orchestration

- **Plan first**: Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions). If something goes sideways, STOP and re-plan — don't keep pushing.
- **Subagents**: Use subagents liberally to keep main context clean. Offload research, exploration, and parallel analysis. One task per subagent.
- **Verify before done**: Never mark a task complete without proving it works. Run tests, check logs, demonstrate correctness. Ask yourself: "Would a staff engineer approve this?"
- **Demand elegance (balanced)**: For non-trivial changes, pause and ask "is there a more elegant way?" Skip this for simple, obvious fixes — don't over-engineer.
- **Autonomous bug fixing**: When given a bug report, just fix it. Point at logs, errors, failing tests — then resolve them. Zero context switching from the user.
- **Simplicity first**: Make every change as simple as possible. Impact minimal code. Find root causes — no temporary fixes.

---

## Best Practices

- SOLID, KISS, idiomatic Go
- One skill at a time, order: Domain -> Application -> Adapters -> Infra
- Error handling with single DomainError (6 codes)
- Ports contain ONLY interfaces
- Testing with manual spies + testify (require/assert)
- Structured logging with zerolog (snake_case, no Sprintf)
- Tracing with OpenTelemetry (spans per layer)

---

## Self-Correction Protocol

When the user corrects you on a convention, pattern, or practice:

1. **Fix the code** immediately as indicated
2. **Identify** which `.claude/docs/back/` file contains (or should contain) that rule
3. **Read** the doc and **update** it with the correction so the mistake never repeats
4. **If critical**: also add to "Important Notes" above
5. **Confirm** to the user: "Correction saved to `.claude/docs/back/{file}` — {what changed}"

This ensures corrections are **persistent** across conversations. The user corrects once, never again.

---

## Available Commands

| Command | Description |
| --- | --- |
| `/create-pr` | Prepare branch, commit, push, and create PR |
| `/pre-pr` | Full pre-PR check (4 agents + fixes) |
| `/scaffold` | Generate entity structure (aggregate, usecase, adapter, etc.) |
| `/review` | Code review focused on branch changes |
| `/test-plan` | Analyze test gaps and optionally write tests |
| `/health-check` | Verify compliance with all conventions |
| `/generate-docs` | Generate complete documentation in `ai_docs/` |
| `/doc-context` | Generate `.claude/` structure for any repository |
