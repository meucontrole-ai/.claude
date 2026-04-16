# Outbox Pattern

## Quando Ativar

- Garantir entrega confiável de eventos para consumers externos (webhook, SQS, SNS)
- Evitar double-publish ou eventos perdidos entre DB commit e message broker
- Implementar o padrão transacional "DB + publish" sem two-phase commit
- Validar flows com LocalStack antes de produção

---

## Problema que resolve

```
// ❌ abordagem ingênua — eventos podem sumir ou duplicar
func (uc *CreatePaymentUseCase) Execute(ctx, cmd) {
    payment := ...
    orderRepo.Save(ctx, payment)      // commit no DB
    eventBus.Publish("payment.created", payment)  // se falhar aqui, evento perdido
}
```

Se o `Publish` falha depois do DB commit, o evento some. Se você faz retry após falha, pode duplicar. A solução: **gravar o evento como linha no MESMO DB em que o agregado foi salvo, dentro da MESMA transação**. Um worker lê o outbox e publica assincronamente, com at-least-once garantido.

---

## Estrutura mínima

### 1. Tabela outbox (mesmo DB do agregado)

```sql
CREATE TABLE outbox (
    id            VARCHAR(36) PRIMARY KEY,
    aggregate_id  VARCHAR(255) NOT NULL,
    event_type    VARCHAR(100) NOT NULL,
    payload       JSONB        NOT NULL,
    created_at    TIMESTAMP    NOT NULL,
    published_at  TIMESTAMP,                       -- NULL = pendente
    attempts      INTEGER      NOT NULL DEFAULT 0,
    last_error    TEXT
);

CREATE INDEX idx_outbox_pending ON outbox(created_at) WHERE published_at IS NULL;
```

### 2. Gravar dentro da mesma transação do domínio

```go
// application/payment/create_payment/usecase.go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := orderRepo.Save(ctx, payment); err != nil {
        return err
    }
    return outboxRepo.Save(ctx, outbox.Entry{
        ID:          shared.GenerateID(),
        AggregateID: payment.ID,
        EventType:   "payment.created",
        Payload:     marshalPayload(payment),
        CreatedAt:   time.Now().UTC(),
    })
})
```

Se o `Publish` externo ainda não aconteceu e o serviço cai, a linha fica lá esperando. Se o Publish acontece duas vezes (retry), o consumer deve ser idempotente (dedupe por `id` do evento).

### 3. Worker de relay

```go
// worker/outbox_relay.go
func (r *OutboxRelay) Run(ctx context.Context) error {
    for {
        entries, err := r.repo.FindPending(ctx, batchSize)
        if err != nil { log.Error(err); continue }

        for _, e := range entries {
            if err := r.publisher.Publish(ctx, e.EventType, e.Payload); err != nil {
                _ = r.repo.RecordFailure(ctx, e.ID, err)
                continue
            }
            _ = r.repo.MarkPublished(ctx, e.ID, time.Now().UTC())
        }

        select {
        case <-ctx.Done():
            return nil
        case <-time.After(pollInterval):
        }
    }
}
```

---

## Publishers possíveis

| Destino | Implementação |
|---|---|
| Webhook HTTP | `webhook.Client` com HMAC signature + retry exponencial |
| SQS/SNS | AWS SDK com `MessageAttributes` pro deduping |
| Kafka | `sarama` com idempotent producer |

Interface comum:

```go
// ports/outbound/event_publisher.go
type EventPublisher interface {
    Publish(ctx context.Context, eventType string, payload []byte) error
}
```

---

## Validação com LocalStack

Antes de subir pra produção, sempre validar end-to-end com LocalStack (SQS/SNS/S3 local):

```yaml
# docker-compose.yml
services:
  localstack:
    image: localstack/localstack:latest
    environment:
      - SERVICES=sqs,sns,s3
    ports: ["4566:4566"]
  app:
    build: .
    environment:
      - AWS_ENDPOINT_URL=http://localstack:4566
      - OUTBOX_PUBLISHER=sqs
    depends_on:
      localstack:
        condition: service_healthy
```

Script de smoke-test após build:

```bash
# start infra
docker compose up -d localstack postgres

# criar fila
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name payments-events

# rodar app
docker compose up -d app

# disparar caso de uso
curl -X POST localhost:8080/payments -d '{...}'

# verificar entrega
aws --endpoint-url=http://localhost:4566 sqs receive-message --queue-url http://localhost:4566/000000000000/payments-events
```

Se a mensagem não chega, diagnosticar pela tabela `outbox`:

```sql
SELECT id, event_type, attempts, last_error, published_at FROM outbox ORDER BY created_at DESC LIMIT 10;
```

---

## Purge de outbox antiga

Entries publicadas podem ser removidas depois de N dias pra não inflar o DB:

```go
// TRUNCATE/DELETE em cron job separado
DELETE FROM outbox WHERE published_at IS NOT NULL AND published_at < NOW() - INTERVAL '7 days';
```

---

## Regras

1. Outbox na MESMA transação do agregado, MESMO DB (não use DB externo).
2. Worker de relay é um processo separado, polling com backoff.
3. Consumer externo DEVE ser idempotente por `event.id`.
4. Falha no Publish incrementa `attempts` e grava `last_error`. Circuit breaker após N tentativas.
5. Validar E2E com LocalStack antes de produção — `docker compose up + curl + check queue`.
6. Monitorar fila de pending: alertar se `COUNT(*) WHERE published_at IS NULL` cresce.
7. Purge de linhas antigas publicadas via cron, não no hot path.
