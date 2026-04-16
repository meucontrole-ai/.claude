# Backend Global Rules

Quick reference for Go backend conventions. Routing is handled by CLAUDE.md — this file contains only global rules that apply across ALL backend skills.

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Functions | PascalCase verb | `CreatePayment`, `FindByID`, `ValidatePayment` |
| Interfaces | PascalCase noun | `PaymentRepository`, `PaymentProvider`, `EventPublisher` |
| Structs | PascalCase noun | `Payment`, `PaymentResult`, `Money`, `UseCase` |
| Packages | lowercase joined | `createpayment`, `postgres`, `handler` |
| Directories | snake_case | `create_payment/`, `service_core/` |
| Files | snake_case | `payment_repository.go`, `usecase.go` |
| Error constants | `ErrCode` prefix | `ErrCodeNotFound`, `ErrCodeConflict` |
| Event constants | `Event` prefix | `EventPaymentCreated` = `"payment.created"` |
| Status constants | `Status` prefix | `StatusCreated` = `"created"` |

## Idiomatic Go

- `error` as last return value. `context.Context` as first parameter
- Composition via embedding (`shared.AggregateRoot` in Payment)
- Small interfaces (1-3 methods preferred)
- **One directory = one package**. NEVER subdirectories of same package
- No `init()` except driver registration
- File > 500 lines: split it
- Godoc mandatory on exported functions/types

## Principles

SOLID, Clean Code, KISS, DRY, YAGNI, Fail Fast. Design Patterns only when solving a real problem.

## Errors

- ONE single type: `DomainError{Code, Message, Err}`
- NEVER return raw infra errors — wrap in DomainError
- 6 codes: `INVALID_ARGUMENT`, `NOT_FOUND`, `CONFLICT`, `INVALID_STATE_TRANSITION`, `INVARIANT_VIOLATION`, `IDEMPOTENCY_CONFLICT`

## Ports

- Contain ONLY interfaces — zero structs, zero DTOs, zero implementation
- Commands and Results live in `domain/`, NOT in `ports/`

## Testing

- Hand-written spies, NOT generated mocks
- Table-driven tests, testify (assert/require)
- `-race` flag mandatory

## Money Handling (CRITICAL)

- Store money as `int64` cents inside `shared.Money{Amount int64, Currency string}`. NEVER float.
- API responses expose cents as-is (`"orderAmount": 2390` = R$ 23,90). The frontend divides by 100 UMA única vez no formatter.
- NEVER expose `amount / 100` in backend responses — double-conversion no cliente = bug garantido (R$ 3,50 virando R$ 350 e vice-versa).
- Arithmetic: use domain methods (`money.Add`, `money.Subtract`) que validam currency.
- Columns must be `BIGINT` / `NUMERIC(20)` — nunca `DECIMAL` ou `FLOAT`.

## Repository Updates (CRITICAL)

- When building the GORM `Updates(map[string]interface{}{...})` map, **list EVERY mutable field of the aggregate**. Forget one (ex: `session_id`) and mutations in memory never reach the DB — the next read returns the old value, a silent data-loss bug.
- Prefer `db.Save(&model)` for full-row updates when you don't need partial column control. `Updates` with maps is only for explicit column-picking with optimistic locking.
- Optimistic locking: always `WHERE id = ? AND version = ?` + bump `version` in the map. Check `RowsAffected == 0` → conflict.
- Golden rule: every field you mutated via a domain method (`t.SessionID = ...`, `t.Status = ...`) must appear in the Updates map.
