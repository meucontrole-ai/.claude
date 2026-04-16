# Testing Patterns

## Quando Ativar
- Escrever testes para use cases
- Criar test doubles (spy, memory repo)
- Estruturar table-driven tests
- Testar validacao e erros

---

## Como Pensar

Testamos USE CASES, nao handlers ou repositorios isoladamente. O use case e a unidade de comportamento. Usamos:
- **Spy publishers**: capturam eventos publicados
- **Memory repos**: implementacoes in-memory dos ports
- **Table-driven tests**: um slice de test cases
- **Testify**: `assert` para verificacoes, `require` para pre-condicoes

NAO usamos mocks gerados (gomock, mockery). Preferimos spies escritos a mao porque sao mais legiveis e nao precisam de geracao de codigo.

---

## Spy Publisher

```go
// Dentro do arquivo de teste (nao compartilhado)
type spyPublisher struct {
    mu     sync.Mutex
    events []saga.ValidationCompletedEvent
    err    error  // Injeta erro para testar falha
}

func (s *spyPublisher) Publish(_ context.Context, event saga.ValidationCompletedEvent) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.events = append(s.events, event)
    return s.err
}

func (s *spyPublisher) lastEvent() saga.ValidationCompletedEvent {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.events[len(s.events)-1]
}
```

Mutex porque testes com `-race` podem detectar race conditions. `err` injetavel para testar cenario de falha.

---

## Helpers de Dados Validos

```go
func validCardPayment() payment.CreatePaymentCommand {
    return payment.CreatePaymentCommand{
        MerchantID:         "merchant-1",
        WorkspaceID:        "ws-1",
        IdempotencyKey:     "idem-1",
        Amount:             10000,
        Currency:           "BRL",
        CaptureMethod:      "automatic",
        ConfirmationMethod: "automatic",
        MethodType:         "card",
        Card: &payment.CardInput{
            Token:           "tok_abc",
            Brand:           "visa",
            LastFourDigits:  "1234",
            ExpirationMonth: 12,
            ExpirationYear:  2026,
            HolderName:      "Bernardo Silva",
            Funding:         "credit",
            Installments:    1,
        },
    }
}

func validPixPayment() payment.CreatePaymentCommand { ... }
func validBoletoPayment() payment.CreatePaymentCommand { ... }
```

Funcoes que retornam dados validos base. Testes de falha modificam um campo:

```go
cmd: func() saga.ValidatePaymentCommand {
    p := validCardPayment()
    p.MerchantID = ""  // Invalida um campo
    return saga.ValidatePaymentCommand{OrderID: "order-6", Payment: p}
}(),
```

---

## Table-Driven Tests

```go
func TestUseCase_Execute(t *testing.T) {
    tests := []struct {
        name         string
        cmd          saga.ValidatePaymentCommand
        wantStatus   string
        wantCode     string
        wantMsgParts []string
    }{
        {
            name: "card valido retorna SUCCESS",
            cmd: saga.ValidatePaymentCommand{
                OrderID: "order-1",
                Payment: validCardPayment(),
            },
            wantStatus: saga.ValidationStatusSuccess,
        },
        {
            name: "merchant_id vazio retorna FAILED",
            cmd: func() saga.ValidatePaymentCommand {
                p := validCardPayment()
                p.MerchantID = ""
                return saga.ValidatePaymentCommand{OrderID: "order-6", Payment: p}
            }(),
            wantStatus: saga.ValidationStatusFailed,
            wantCode:   "INVALID_ARGUMENT",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            pub := &spyPublisher{}
            uc := NewUseCase(pub)

            err := uc.Execute(context.Background(), tt.cmd)
            require.NoError(t, err)

            event := pub.lastEvent()
            assert.Equal(t, tt.wantStatus, event.Status)

            if tt.wantCode != "" {
                assert.Equal(t, tt.wantCode, event.ErrorCode)
            }

            for _, part := range tt.wantMsgParts {
                assert.True(t,
                    strings.Contains(strings.ToLower(event.ErrorMessage), strings.ToLower(part)),
                    "error_message deveria conter %q, got: %q", part, event.ErrorMessage,
                )
            }
        })
    }
}
```

---

## Assert Helper com t.Helper()

```go
func assertFixedFields(t *testing.T, event saga.ValidationCompletedEvent, orderID string) {
    t.Helper()
    _, err := uuid.Parse(event.EventID)
    assert.NoError(t, err, "event_id deve ser UUID valido")
    assert.Equal(t, "validation.payment.completed", event.EventType)
    assert.Equal(t, "payment-service", event.Source)
    assert.Equal(t, orderID, event.OrderID)
}
```

`t.Helper()` faz o erro apontar para o caller, nao para o helper.

---

## Teste de Erro do Publisher

```go
func TestUseCase_Execute_PublisherError(t *testing.T) {
    pub := &spyPublisher{err: assert.AnError}  // Injeta erro
    uc := NewUseCase(pub)

    err := uc.Execute(context.Background(), saga.ValidatePaymentCommand{
        OrderID: "order-1",
        Payment: validCardPayment(),
    })
    require.ErrorIs(t, err, assert.AnError)
}
```

Teste separado (nao table-driven) para cenarios de infra error.

---

## Convencoes

- Nomes de teste em portugues: `"card valido retorna SUCCESS"`, `"merchant_id vazio retorna FAILED"`
- Funcao de teste: `TestUseCase_Execute` (struct_metodo)
- Helpers privados: `validCardPayment()`, `assertFixedFields()`
- IIFEs para setup inline: `cmd: func() { ... }()`
- `require.NoError` para pre-condicoes, `assert.Equal` para verificacoes

---

## Regras

1. Spies escritos a mao, NAO mocks gerados
2. Spy com `sync.Mutex` para `-race` safety
3. Table-driven tests para variacoes de input/output
4. Testes de erro de infra separados (nao table-driven)
5. Helpers para dados validos (`validCardPayment()`)
6. `t.Helper()` em todo assert helper
7. `require.NoError` para erros fatais, `assert.Equal` para verificacoes
8. Flag `-race` obrigatoria: `go test ./... -race`
9. Testes no mesmo package do use case (nao _test package separado)

---

## Teste de regressão de state machine

Bugs de state machine são silenciosos — código que "quase funciona" passa code review. Pro contrato da máquina de estado ficar trancado, teste **TODAS** as transições válidas E inválidas:

```go
func TestStatusTransitions(t *testing.T) {
    all := []Status{StatusCreated, StatusPending, StatusFinal, StatusConcluded, StatusCancelled}

    for _, from := range all {
        for _, to := range all {
            t.Run(fmt.Sprintf("%s → %s", from, to), func(t *testing.T) {
                got := from.CanTransitionTo(to)
                want := expectedTransition(from, to)
                assert.Equal(t, want, got)
            })
        }
    }
}
```

Se alguém adicionar um status novo ou mudar `validTransitions`, o teste quebra com diff claro mostrando todas as células afetadas.

### Teste de reversão (Undo)

Se o domínio tem operações revertíveis, teste que o método aceita o status "reversível":

```go
func TestUndoAcceptsFinal(t *testing.T) {
    a := aggregateWithStatus(StatusFinal)
    err := a.Undo()
    assert.NoError(t, err)
    assert.Equal(t, StatusPrevious, a.Status)
}
```

Garante que `IsTerminal()` não bloqueia a reversão inadvertidamente.

---

## Teste de idempotência

Operações idempotentes devem manter identidade (SessionID, correlation) entre chamadas repetidas:

```go
func TestActivateTwiceKeepsSession(t *testing.T) {
    a := NewAggregate()
    _ = a.Activate(attrs1)
    first := a.SessionID

    _ = a.Activate(attrs2)
    assert.Equal(t, first, a.SessionID)
}
```

---

## Integration tests com LocalStack

Use cases que interagem com SQS/SNS/S3 devem ter integration test com LocalStack:

```bash
docker compose -f docker-compose.test.yml up -d localstack postgres
go test ./... -tags=integration
docker compose -f docker-compose.test.yml down
```

Marker de build tag (`//go:build integration`) isola esses testes do ciclo rápido de unit tests.
