# State Machine

## Quando Ativar
- Adicionar novo status de pagamento
- Modificar transicoes validas
- Entender guards de estado
- Trabalhar com ApplyProviderStatus (webhooks)

---

## Como Pensar

A maquina de estados e a lei fundamental do pagamento. Nenhum codigo — nem use case, nem webhook, nem migration — pode violar as transicoes definidas em `validTransitions`.

Pense como: "o Status e um semaforo. `validTransitions` e o circuito que controla quando muda. Ninguem pula o circuito."

Se um provider reporta um status inesperado, o pagamento NAO transiciona — a funcao retorna erro. Isso e intencional.

---

## Status e Transicoes

```go
// internal/domain/payment/status.go
type Status string

const (
    StatusCreated           Status = "created"
    StatusPending           Status = "pending"
    StatusAuthorized        Status = "authorized"
    StatusCaptured          Status = "captured"
    StatusFailed            Status = "failed"
    StatusCanceled          Status = "canceled"
    StatusRefunded          Status = "refunded"
    StatusPartiallyRefunded Status = "partially_refunded"
    StatusExpired           Status = "expired"
)

var validTransitions = map[Status]map[Status]bool{
    StatusCreated: {
        StatusPending:  true,
        StatusFailed:   true,
        StatusCanceled: true,
    },
    StatusPending: {
        StatusAuthorized: true,
        StatusCaptured:   true,
        StatusFailed:     true,
        StatusCanceled:   true,
        StatusExpired:    true,
    },
    StatusAuthorized: {
        StatusCaptured: true,
        StatusCanceled: true,
        StatusFailed:   true,
        StatusExpired:  true,
    },
    StatusCaptured: {
        StatusRefunded:          true,
        StatusPartiallyRefunded: true,
    },
    StatusPartiallyRefunded: {
        StatusRefunded:          true,
        StatusPartiallyRefunded: true, // Permite multiplos estornos parciais
    },
}
```

IMPORTANTE: O tipo e `map[Status]map[Status]bool` (nao `map[Status][]Status`).

Lookup e O(1): `validTransitions[current][target]` retorna `true` ou `false`.

---

## Guards de Estado

```go
func (s Status) CanTransitionTo(target Status) bool {
    targets, ok := validTransitions[s]
    if !ok {
        return false  // Status terminal — nenhuma transicao valida
    }
    return targets[target]
}

func (s Status) IsTerminal() bool {
    _, hasTransitions := validTransitions[s]
    return !hasTransitions
    // Terminal: failed, canceled, refunded, expired
}

func (s Status) IsCapturable() bool {
    return s == StatusAuthorized
}

func (s Status) IsRefundable() bool {
    return s == StatusCaptured || s == StatusPartiallyRefunded
}

func (s Status) IsCancelable() bool {
    return s == StatusCreated || s == StatusPending || s == StatusAuthorized
}
```

Os metodos `Is*` sao convenience — o agregado os usa para validacao semantica antes de `transitionTo`.

---

## Diagrama de Transicoes

```
created → pending → authorized → captured → partially_refunded → refunded
  ↓         ↓          ↓                            ↓
 failed   failed     failed                   partially_refunded
  ↓         ↓          ↓
canceled  canceled   canceled
            ↓          ↓
          expired    expired
            ↓
          captured (direto, via provider)
```

---

## ProviderStatus (Abstracão de Webhook)

```go
type ProviderStatus string

const (
    ProviderStatusAuthorized ProviderStatus = "authorized"
    ProviderStatusCaptured   ProviderStatus = "captured"
    ProviderStatusRefunded   ProviderStatus = "refunded"
    ProviderStatusFailed     ProviderStatus = "failed"
)
```

`ProviderStatus` abstrai os status especificos de cada provider (Matera, Stripe, etc.) em conceitos de dominio. O adapter de webhook converte: status do provider → `ProviderStatus` → `ApplyProviderStatus()`.

---

## Como Adicionar Novo Status

1. Adicione a constante em `status.go`
2. Adicione as transicoes em `validTransitions`
3. Se precisar de guard, adicione metodo `Is*()` no `Status`
4. Adicione o metodo de mutacao no `Payment` (ex: `MarkPending()`)
5. Crie o evento de dominio em `events.go`
6. Se vier de webhook, adicione o case em `ApplyProviderStatus`

---

## Regras

1. `validTransitions` e a UNICA fonte de verdade para transicoes
2. Status terminal = nao tem entrada em `validTransitions`
3. `CanTransitionTo` e O(1) lookup, nao iteracao
4. `transitionTo()` e PRIVADO — chamado apenas pelos metodos do agregado
5. Webhooks usam `ApplyProviderStatus`, nao `transitionTo` diretamente
6. `StatusPartiallyRefunded` → `StatusPartiallyRefunded` e valido (multiplos estornos parciais)
7. `StatusPending` → `StatusCaptured` e valido (providers que pulam autorizacao)
