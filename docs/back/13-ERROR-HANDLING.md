# Error Handling

## Quando Ativar
- Criar ou propagar erros
- Converter erros entre camadas
- Mapear erros para codigos gRPC
- Decidir entre NewDomainError e WrapDomainError

---

## Como Pensar

Um unico tipo de erro: `DomainError`. Nao temos `AppError`, `InfraError`, `HandlerError`. Tudo e DomainError com um Code que indica a categoria.

Isso simplifica a propagacao: qualquer camada pode criar ou wrappear um DomainError, e o handler no topo sabe como mapear para gRPC/HTTP.

---

## DomainError

```go
// internal/domain/shared/errors.go
type DomainError struct {
    Code    string  // Codigo programatico
    Message string  // Mensagem legivel
    Err     error   // Erro original (wrap)
}

func (e *DomainError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *DomainError) Unwrap() error { return e.Err }
```

---

## Construtores de Erro

```go
// Erro simples (sem causa)
NewDomainError(shared.ErrCodeInvalidArgument, "merchant_id is required")

// Erro com causa (wrap)
WrapDomainError(shared.ErrCodeInvalidArgument, "invalid amount/currency", err)
```

Use `NewDomainError` quando voce detecta o problema. Use `WrapDomainError` quando outra funcao retornou erro e voce quer adicionar contexto.

---

## Codigos de Erro

```go
const (
    ErrCodeInvalidArgument     = "INVALID_ARGUMENT"       // Input invalido
    ErrCodeNotFound            = "NOT_FOUND"               // Recurso nao existe
    ErrCodeConflict            = "CONFLICT"                // Conflito de unicidade ou versao
    ErrCodeInvalidTransition   = "INVALID_STATE_TRANSITION" // Transicao de status invalida
    ErrCodeInvariantViolation  = "INVARIANT_VIOLATION"     // Regra de negocio violada
    ErrCodeIdempotencyConflict = "IDEMPOTENCY_CONFLICT"    // Chave de idempotencia duplicada
)
```

---

## Onde Cada Erro e Criado

| Codigo | Quem cria | Exemplo |
|---|---|---|
| INVALID_ARGUMENT | Domain construtores | `NewPayment()` com campo vazio |
| NOT_FOUND | Repository | `FindByID()` sem resultado |
| CONFLICT | Repository | Duplicate key, optimistic lock |
| INVALID_STATE_TRANSITION | Domain `transitionTo()` | created → refunded |
| INVARIANT_VIOLATION | Domain + Repository | Capture > authorized amount |
| IDEMPOTENCY_CONFLICT | IdempotencyStore | Chave duplicada |

---

## Propagacao entre Camadas

```
Domain: NewDomainError(ErrCodeInvalidArgument, "merchant_id is required")
   ↓
Application: return nil, err  (propaga sem modificar)
   ↓
Adapter (gRPC): return nil, status.Error(codes.Internal, err.Error())
```

O use case propaga o erro SEM wrappear (a menos que adicione contexto). O handler converte para gRPC status.

---

## Erros de Infra no Repositorio

```go
// Repositorio SEMPRE converte erros de infra para DomainError:
if result.Error != nil {
    if isDuplicateKeyError(result.Error) {
        return shared.WrapDomainError(shared.ErrCodeConflict, "...", result.Error)
    }
    return shared.WrapDomainError(shared.ErrCodeInvariantViolation, "failed to save", result.Error)
}
```

NUNCA retorne erro GORM puro para camadas superiores.

---

## Regras

1. UM unico tipo de erro: `DomainError` — nao crie tipos novos
2. `NewDomainError` para erros que voce detecta
3. `WrapDomainError` para erros que voce reenvia com contexto
4. NUNCA retorne erros de infra puros (GORM, SQL, HTTP) — wrapeie em DomainError
5. Use codes.As/errors.As para extrair DomainError quando necessario
6. 6 codigos de erro — nao invente novos sem necessidade
7. Message em ingles, sem dados sensiveis (PII)
