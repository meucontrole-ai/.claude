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

1. `validTransitions` é a ÚNICA fonte de verdade para transições
2. Status terminal = não tem entrada em `validTransitions`
3. `CanTransitionTo` é O(1) lookup, não iteração
4. `transitionTo()` é PRIVADO — chamado apenas pelos métodos do agregado
5. Webhooks usam `ApplyProviderStatus`, não `transitionTo` diretamente
6. `StatusPartiallyRefunded` → `StatusPartiallyRefunded` é válido (múltiplos estornos parciais)
7. `StatusPending` → `StatusCaptured` é válido (providers que pulam autorização)

---

## Armadilhas

### 1. Guard `IsTerminal()` pode tornar código revertível inalcançável

Se uma operação existe pra reverter um status (ex: um `Undo*` que volta `FinalState → PreviousState`), esse status **não pode** ser tratado como terminal na guarda de entrada do método — senão o branch de reversão fica morto.

```go
// ❌ errado
func (s Status) IsTerminal() bool {
    return s == StatusFinal || s == StatusConcluded || s == StatusCancelled
}
func (a *Aggregate) Undo() error {
    if a.Status.IsTerminal() {
        return error  // rejeita StatusFinal, impossibilita reversão
    }
    if a.Status == StatusFinal {
        _ = a.transitionTo(StatusPrevious)  // código inalcançável
    }
}

// ✅ correto — guarda checa só os realmente irreversíveis
func (a *Aggregate) Undo() error {
    if a.Status == StatusConcluded || a.Status == StatusCancelled {
        return error
    }
    // StatusFinal aceito — método reverte via transitionTo
}
```

Regra: "terminal" pra state machine (sem entrada em `validTransitions`) NÃO é o mesmo que "terminal pra operação X". Cada método revertível define seus próprios inválidos.

### 2. Operações idempotentes preservam identidade de sessão

Se um método pode ser chamado várias vezes no mesmo estado (ex: reattach, update de atributos sem trocar ciclo de vida) e o agregado tem um `SessionID` / correlation ID, **não regenere** quando o recurso já está no estado ativo:

```go
func (a *Aggregate) Activate(attrs Attrs) error {
    isFreshActivation := a.Status != StatusActive || a.SessionID == ""
    a.Attrs = attrs
    a.Status = StatusActive
    if isFreshActivation {
        a.SessionID = shared.GenerateID()
    }
    // já ativo → mantém SessionID → audit log não fragmenta
    return nil
}
```

Senão cada chamada gera um ID novo e o audit log aparece quebrado em grupos separados.

### 3. `RecordEvent` ≠ persistência no event_log

`a.RecordEvent(...)` anexa o evento de domínio ao agregado. `a.PullEvents()` só retira pra consumo. A **persistência** é responsabilidade do use case chamar `eventLogger.Log(LogParams{...})`. Esquecer = estado muda, audit log não registra.

### 4. SessionID top-level no LogParams, não só no diff

Sistemas que agrupam eventos por sessão precisam do ID no **campo dedicado** do event_log (coluna `session_id`), não só dentro do JSON de `diff`. Camadas que consultam (frontend, timeline) podem usar um ou outro, e divergência = agrupamento quebrado.

Regra: quando o use case roda em contexto de sessão, popule **ambos** — `LogParams.SessionID` E `diff["session_id"]`.
