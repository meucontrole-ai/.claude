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
