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

---

## Outbox pattern como pré-requisito

Pra garantir que eventos disparados do domínio **cheguem** ao consumer externo (SQS/SNS/webhook), use o padrão outbox (docs/back/28-OUTBOX-PATTERN.md):

1. Gravar evento como linha em `outbox` dentro da MESMA transação do agregado.
2. Worker separado lê `WHERE published_at IS NULL`, publica, marca `published_at`.
3. Consumer externo idempotente por `event.id`.

Sem outbox, `DB commit + publish` não é atômico e eventos podem sumir ou duplicar.

---

## Validação E2E com LocalStack

Antes de produção, valide outbox → SQS → consumer em ambiente local:

```yaml
# docker-compose.test.yml
services:
  localstack:
    image: localstack/localstack:latest
    environment: [SERVICES=sqs,sns]
    ports: ["4566:4566"]
```

```bash
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name events
curl -X POST localhost:8080/...   # trigger use case
aws --endpoint-url=http://localhost:4566 sqs receive-message --queue-url ...
```

Se mensagem não chega, diagnóstico pela tabela `outbox`:
```sql
SELECT event_type, attempts, last_error, published_at
FROM outbox ORDER BY created_at DESC LIMIT 10;
```

---

## Consumer idempotente

At-least-once delivery = mesma mensagem pode chegar 2x. Consumer não pode duplicar efeito:

### 1. Dedupe por event ID

```go
func (h *Handler) Handle(ctx context.Context, msg Message) error {
    if already, _ := h.store.Exists(ctx, msg.EventID); already {
        return nil
    }
    if err := h.doBusiness(ctx, msg); err != nil {
        return err
    }
    return h.store.MarkProcessed(ctx, msg.EventID)
}
```

### 2. Idempotência natural do estado

Se a operação em si é idempotente (ex: `SET status = 'paid' WHERE id = X`), rodar 2x é inofensivo. Preferir essa abordagem quando possível.

---

## Graceful shutdown

Consumers em produção DEVEM terminar mensagem em andamento antes de encerrar:

```go
func (c *Consumer) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            c.waitInFlightDrain()
            return nil
        default:
            msg := c.queue.Receive(ctx)
            c.handle(msg)
        }
    }
}
```

Senão, `SIGTERM` no meio do processamento deixa mensagem "em voo" que vai ser re-entregue → duplicação.
