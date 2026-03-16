# Messaging Service (Saga)

## Quando Ativar
- Trabalhar com SQS consumer
- Implementar saga de validacao
- Modificar consumer in-memory
- Entender pipeline de mensageria

---

## Como Pensar

O messaging service consome mensagens de fila (SQS em prod, memory em dev) e executa validacoes. E o padrao saga: cada servico valida sua parte e publica o resultado.

Nao confunda com event publishing (domain events). Messaging e para saga orchestration — comunicacao entre servicos.

---

## Estrutura

```
internal/adapters/messaging/
  handler/
    validation_handler.go      # Processa mensagem → chama use case
    validation_handler_test.go
  dto/
    saga_validation_request.go # DTO de deserializacao SQS
  memory/
    consumer.go                # Consumer in-memory (dev/test)
    consumer_test.go
  sqs/
    consumer.go                # Consumer SQS (producao)
```

---

## Consumer Pattern

```go
// Consumidor recebe mensagem, deserializa, chama handler
type Consumer interface {
    Start(ctx context.Context) error
    Stop() error
}
```

Dois adapters:
- `memory.Consumer`: canal Go, para dev/test
- `sqs.Consumer`: polling SQS real, para producao

O handler de validacao e o mesmo — injetado no consumer.

---

## Saga Domain

```go
// internal/domain/saga/command.go
type ValidatePaymentCommand struct {
    OrderID     string
    Payment     payment.CreatePaymentCommand
    Transaction *TransactionInput
}

// internal/domain/saga/event.go
type ValidationCompletedEvent struct {
    EventID           string
    EventType         string  // "validation.payment.completed"
    Source            string  // "payment-service"
    OrderID           string
    ValidationService string  // "payment"
    Status            string  // SUCCESS ou FAILED
    ErrorCode         string
    ErrorMessage      string
    // ...
}
```

---

## Regras

1. Consumer e adapter, handler e adapter — use case e application
2. SQS consumer para producao, memory consumer para dev/test
3. Saga domain (commands/events) vive em `domain/saga/`
4. Consumer implementa graceful shutdown via context
5. Desserializacao de mensagem SQS → DTO → domain command
