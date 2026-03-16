# Aggregates

## Quando Ativar
- Criar ou modificar aggregate root
- Adicionar campo ou metodo a Payment
- Trabalhar com ciclo de vida do agregado
- Entender construtores e mutacoes

---

## Como Pensar

Aggregate e a unidade de consistencia. Tudo dentro do agregado e consistente a todo momento — invariantes nunca sao violadas.

Princípios fundamentais:
1. **Construtor valida tudo**: se `NewPayment()` retornou, o Payment e valido
2. **Mutacoes sao verbos de negocio**: `Authorize()`, `Capture()`, nao `SetStatus()`
3. **Estado interno e protegido**: `transitionTo()` e `touch()` sao privados
4. **Eventos registram o que aconteceu**: cada transicao grava um evento

Nao pense "vou mudar o campo Status". Pense "vou autorizar o pagamento" — o metodo `Authorize()` cuida de todos os efeitos colaterais.

---

## Codigo Real: Payment Aggregate

```go
// internal/domain/payment/payment.go
type Payment struct {
    shared.AggregateRoot  // Embutido — fornece RecordEvent, PullEvents

    ID             string
    MerchantID     string
    WorkspaceID    string
    IdempotencyKey string

    Amount         shared.Money
    CapturedAmount shared.Money
    RefundedAmount shared.Money

    Status             Status
    CaptureMethod      CaptureMethod
    ConfirmationMethod ConfirmationMethod

    Method  Method
    Billing BillingDetails

    ProviderName          string
    ProviderTransactionID string
    ProviderMetadata      shared.Metadata

    Attempts []Attempt

    Metadata    shared.Metadata
    Description string

    AuthorizedAt *time.Time
    CapturedAt   *time.Time
    FailedAt     *time.Time
    CanceledAt   *time.Time
    ExpiredAt    *time.Time
    CreatedAt    time.Time
    UpdatedAt    time.Time
    Version      int        // Optimistic locking
}
```

Note:
- `shared.AggregateRoot` embutido (nao campo nomeado)
- Campos exportados (PascalCase) — o domain e transparente para a application layer
- `Version` para optimistic locking
- Timestamps com ponteiro (`*time.Time`) para campos opcionais
- `CreatedAt` e `UpdatedAt` sem ponteiro — sempre preenchidos

---

## Construtor: Fail-Fast Validation

```go
type CreateParams struct {
    MerchantID         string
    WorkspaceID        string
    IdempotencyKey     string
    Amount             shared.Money
    CaptureMethod      CaptureMethod
    ConfirmationMethod ConfirmationMethod
    Method             Method
    Billing            BillingDetails
    Description        string
    Metadata           shared.Metadata
}

func NewPayment(params CreateParams) (*Payment, error) {
    if params.MerchantID == "" {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "merchant_id is required")
    }
    if params.IdempotencyKey == "" {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "idempotency_key is required")
    }
    if !params.Amount.IsPositive() {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "amount must be positive")
    }
    if !params.CaptureMethod.IsValid() {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "invalid capture method")
    }
    if !params.ConfirmationMethod.IsValid() {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "invalid confirmation method")
    }
    if err := params.Method.Validate(); err != nil {
        return nil, shared.WrapDomainError(shared.ErrCodeInvalidArgument, "invalid payment method", err)
    }
    if params.Metadata == nil {
        params.Metadata = shared.NewMetadata()
    }

    now := time.Now().UTC()
    p := &Payment{
        ID:                 shared.GenerateID(),
        MerchantID:         params.MerchantID,
        // ... todos os campos ...
        Status:             StatusCreated,
        Attempts:           make([]Attempt, 0),
        CreatedAt:          now,
        UpdatedAt:          now,
        Version:            1,
    }

    p.RecordEvent(PaymentCreatedEvent{
        BaseEvent: shared.BaseEvent{
            ID:            shared.GenerateID(),
            Type:          EventPaymentCreated,
            AggregateRef:  p.ID,
            AggregateKind: "payment",
            Timestamp:     now,
        },
        MerchantID:     p.MerchantID,
        Amount:         p.Amount,
        MethodType:     p.Method.Type,
        IdempotencyKey: p.IdempotencyKey,
    })

    return p, nil
}
```

Padrao: `CreateParams` como parameter object → validacao fail-fast → inicializa → grava evento → retorna.

---

## Mutacoes: Verbos de Negocio

```go
// Authorize transiciona o pagamento para autorizado.
func (p *Payment) Authorize(providerName, providerTxnID string) error {
    if err := p.transitionTo(StatusAuthorized); err != nil {
        return err
    }
    now := time.Now().UTC()
    p.ProviderName = providerName
    p.ProviderTransactionID = providerTxnID
    p.AuthorizedAt = &now
    p.touch()

    p.RecordEvent(PaymentAuthorizedEvent{...})
    return nil
}

// Capture captura o valor autorizado (suporta captura parcial).
func (p *Payment) Capture(captureAmount shared.Money) error {
    if !p.Status.IsCapturable() {
        return shared.NewDomainError(shared.ErrCodeInvalidTransition, ...)
    }
    if captureAmount.Currency != p.Amount.Currency {
        return shared.NewDomainError(shared.ErrCodeInvalidArgument, "capture currency mismatch")
    }
    if captureAmount.Amount <= 0 || captureAmount.Amount > p.Amount.Amount {
        return shared.NewDomainError(shared.ErrCodeInvariantViolation, ...)
    }

    if err := p.transitionTo(StatusCaptured); err != nil {
        return err
    }
    // ... atualiza campos, touch, RecordEvent
    return nil
}
```

Padrao de cada mutacao:
1. Valida pre-condicoes (status guards, invariantes de negocio)
2. Transiciona estado via `transitionTo()`
3. Atualiza campos relevantes
4. Chama `touch()` (UpdatedAt + Version++)
5. Registra evento de dominio

---

## Metodos Privados do Agregado

```go
// transitionTo validates and applies a status transition.
func (p *Payment) transitionTo(target Status) error {
    if !p.Status.CanTransitionTo(target) {
        return shared.NewDomainError(
            shared.ErrCodeInvalidTransition,
            fmt.Sprintf("invalid transition from %s to %s", p.Status, target),
        )
    }
    p.Status = target
    return nil
}

// touch updates the timestamp and increments version for optimistic locking.
func (p *Payment) touch() {
    p.UpdatedAt = time.Now().UTC()
    p.Version++
}
```

`transitionTo` e `touch` sao PRIVADOS — nenhum codigo externo muda status ou version diretamente.

---

## ApplyProviderStatus: Webhook Idempotente

```go
func (p *Payment) ApplyProviderStatus(params ApplyProviderStatusParams) error {
    switch params.ProviderStatus {
    case ProviderStatusCaptured:
        if p.Status == StatusCaptured {
            return nil  // Idempotente: ja capturado
        }
        if p.Status == StatusAuthorized {
            return p.Capture(p.Amount)
        }
        if p.Status == StatusPending || p.Status == StatusCreated {
            // Multi-step: autoriza primeiro, depois captura
            if err := p.Authorize(params.ProviderName, params.ProviderTxnID); err != nil {
                return err
            }
            return p.Capture(p.Amount)
        }
    // ... outros cases
    }
    return nil
}
```

Trade-off importante: `ApplyProviderStatus` faz transicoes multi-step porque providers podem pular etapas (ex: Matera reporta "captured" direto, sem "authorized" separado).

---

## AggregateRoot Base

```go
// internal/domain/shared/events.go
type AggregateRoot struct {
    events []DomainEvent  // campo privado, nao exportado
}

func (a *AggregateRoot) RecordEvent(event DomainEvent) {
    a.events = append(a.events, event)
}

func (a *AggregateRoot) PullEvents() []DomainEvent {
    events := a.events
    a.events = nil  // Limpa apos pull — cada evento e publicado uma unica vez
    return events
}
```

Fluxo: agregado grava eventos → use case persiste → use case chama PullEvents → publica.

---

## Regras

1. Construtor (`NewPayment`) SEMPRE valida todas as invariantes
2. Mutacoes usam nomes de negocio: `Authorize`, `Capture`, `Cancel`, `Fail`, `Expire`
3. NUNCA mude `Status` diretamente — use `transitionTo()` (privado)
4. NUNCA mude `Version` diretamente — use `touch()` (privado)
5. Todo metodo de mutacao chama `touch()` no final
6. Transicoes de estado SEMPRE gravam evento de dominio
7. `ApplyProviderStatus` lida com idempotencia de webhooks
8. `CreateParams` como parameter object (nao 15 parametros soltos)
9. Campos `*time.Time` para timestamps opcionais, `time.Time` para obrigatorios
