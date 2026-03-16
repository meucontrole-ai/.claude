# Domain Events

## Quando Ativar
- Criar novo evento de dominio
- Modificar publicacao de eventos
- Entender fluxo evento → publicacao

---

## Como Pensar

Eventos registram fatos que aconteceram. Nao sao comandos ("faca X"), sao notificacoes ("X aconteceu"). O agregado grava; o use case publica apos persistir.

Sequencia critica:
1. Agregado executa mutacao
2. Agregado grava evento (`RecordEvent`)
3. Use case salva no banco
4. Use case chama `PullEvents()` e publica

Se publicar ANTES de salvar, o sistema fica inconsistente (evento publicado, dado nao salvo).

---

## Interface e Base

```go
// internal/domain/shared/events.go
type DomainEvent interface {
    EventType() string
    OccurredAt() time.Time
    AggregateID() string
    AggregateType() string
}

type BaseEvent struct {
    ID            string
    Type          string
    AggregateRef  string
    AggregateKind string
    Timestamp     time.Time
}

func (e BaseEvent) EventType() string      { return e.Type }
func (e BaseEvent) OccurredAt() time.Time  { return e.Timestamp }
func (e BaseEvent) AggregateID() string    { return e.AggregateRef }
func (e BaseEvent) AggregateType() string  { return e.AggregateKind }
```

Todos os eventos embeddam `BaseEvent` e adicionam campos especificos.

---

## Eventos de Payment

```go
// internal/domain/payment/events.go
const (
    EventPaymentCreated    = "payment.created"
    EventPaymentAuthorized = "payment.authorized"
    EventPaymentCaptured   = "payment.captured"
    EventPaymentFailed     = "payment.failed"
    EventPaymentCanceled   = "payment.canceled"
    EventPaymentRefunded   = "payment.refunded"
    EventPaymentExpired    = "payment.expired"
)

type PaymentCreatedEvent struct {
    shared.BaseEvent                    // Embeddado, nao campo nomeado
    MerchantID     string
    Amount         shared.Money
    MethodType     MethodType
    IdempotencyKey string
}

type PaymentAuthorizedEvent struct {
    shared.BaseEvent
    ProviderName          string
    ProviderTransactionID string
    Amount                shared.Money
}

type PaymentCapturedEvent struct {
    shared.BaseEvent
    CapturedAmount shared.Money
}

type PaymentFailedEvent struct {
    shared.BaseEvent
    ErrorCode    string
    ErrorMessage string
}

type PaymentCanceledEvent struct {
    shared.BaseEvent
    Reason string
}

type PaymentRefundedEvent struct {
    shared.BaseEvent
    RefundID       string
    RefundedAmount shared.Money
    TotalRefunded  shared.Money
    IsFullRefund   bool
}
```

Convencao:
- Constante: `Event` + agregado + verbo passado (`EventPaymentCreated`)
- Valor: `"payment.created"` (lowercase, dot-separated)
- Struct: agregado + verbo + `Event` (`PaymentCreatedEvent`)
- Campos: apenas dados relevantes para consumidores, nao o agregado inteiro

---

## Como o Agregado Grava Evento

```go
// Dentro de payment.go, no construtor:
p.RecordEvent(PaymentCreatedEvent{
    BaseEvent: shared.BaseEvent{
        ID:            shared.GenerateID(),
        Type:          EventPaymentCreated,
        AggregateRef:  p.ID,
        AggregateKind: "payment",
        Timestamp:     now,
    },
    MerchantID: p.MerchantID,
    Amount:     p.Amount,
    // ...
})
```

Padrao: `shared.BaseEvent` literal com todos os campos preenchidos.

---

## Como o Use Case Publica

```go
// Dentro do use case, APOS persistir:
if err := uc.paymentRepo.Save(ctx, p); err != nil {
    return nil, err
}

events := p.PullEvents()
if len(events) > 0 {
    if err := uc.eventPub.Publish(ctx, events...); err != nil {
        _ = err  // Log, nao falha — at-least-once semantics
    }
}
```

Trade-off: falha na publicacao NAO reverte a persistencia. Isso e intencional — preferimos consistencia eventual a perder o pagamento.

---

## Regras

1. Eventos sao registrados DENTRO do agregado, nas mutacoes
2. Publicacao acontece APOS persistencia bem-sucedida
3. `PullEvents()` limpa a lista — cada evento e publicado uma vez
4. BaseEvent SEMPRE embeddado (nao campo nomeado)
5. `AggregateKind` e sempre o nome do agregado em lowercase ("payment")
6. Falha na publicacao e logada, nao reverte a operacao
7. Cada evento tem ID proprio (uuid) — rastreabilidade
