# Logging Estruturado

## Quando Ativar
- Adicionar, modificar ou revisar logs em qualquer camada
- Configurar logger (nivel, formato, output)
- Garantir que dados sensiveis nao vazem em logs
- Correlacionar logs com traces distribuidos
- Implementar middleware de logging

---

## Como Pensar

Logs sao para **operacao em producao**, nao para debugging local. Cada log deve responder tres perguntas: "o que aconteceu?", "para qual entidade?", "com qual resultado?"

### Decision Tree: Preciso de um Log Aqui?

```
Evento aconteceu?
├── Operacao de negocio (entrada/saida de use case) → SIM, Info
├── Erro recuperavel ou irrecuperavel → SIM, Error com .Err(err)
├── Chamada externa (HTTP, gRPC, DB) → SIM, via middleware/interceptor
├── Loop interno ou iteracao → NAO (poluicao)
├── Debugging temporario → Debug level, remover antes de merge
└── Dado sensivel envolvido → NUNCA logar o dado, apenas o identificador
```

### Trade-offs

| Decisao | Pro | Contra |
|---------|-----|--------|
| JSON em producao | Parseavel por ferramentas (Loki, ELK, CloudWatch) | Ilegivel para humanos |
| Console em dev | Legivel, rapido para debuggar | Nao parseavel |
| Caller (.Caller()) | Saber exatamente onde o log foi emitido | Overhead de reflexao (minimo) |
| Log por middleware | Consistencia, zero esquecimento | Menos contexto de negocio |
| Log manual no use case | Contexto rico de negocio | Risco de inconsistencia |
| **Ambos** | Middleware para request/response + manual para marcos de negocio | Dois pontos de manutencao |

**Resposta correta:** Middleware para o envelope (request in / response out com duracao) + logs manuais para marcos de negocio dentro do use case (max 2-3 por operacao).

---

## Stack e Inicializacao

### Logger Contextual

O logger vive no `context.Context`. Uma funcao wrapper extrai o logger do contexto e injeta campos de correlacao (trace_id, span_id) automaticamente.

```go
// pkg/logger/logger.go

func New(environment string) zerolog.Logger {
    if environment == "production" {
        zerolog.TimeFieldFormat = time.RFC3339Nano
        return zerolog.New(os.Stdout).
            Level(zerolog.InfoLevel).
            With().
            Timestamp().
            Caller().
            Logger()
    }
    return zerolog.New(zerolog.ConsoleWriter{Out: os.Stdout, TimeFormat: time.Kitchen}).
        Level(zerolog.DebugLevel).
        With().
        Timestamp().
        Logger()
}
```

**Regras de inicializacao:**
- Production: JSON stdout, InfoLevel, Timestamp RFC3339Nano, Caller habilitado
- Development: ConsoleWriter stdout, DebugLevel, Timestamp Kitchen, sem Caller

### Registro Global (main)

```go
log := logger.New(cfg.Server.Environment)
zerolog.DefaultContextLogger = &log
```

Todo `main.go` deve registrar o logger como `DefaultContextLogger`. Isso garante fallback quando o context nao contem logger (ex: goroutines desacopladas).

---

## Correlacao com OpenTelemetry

O wrapper `Ctx(ctx)` injeta `trace_id` e `span_id` automaticamente quando um span OTel esta ativo no contexto:

```go
func Ctx(ctx context.Context) zerolog.Logger {
    span := trace.SpanFromContext(ctx)
    sc := span.SpanContext()

    l := zerolog.Ctx(ctx)
    if l.GetLevel() == zerolog.Disabled {
        if zerolog.DefaultContextLogger != nil {
            l = zerolog.DefaultContextLogger
        } else {
            nop := zerolog.Nop()
            l = &nop
        }
    }

    if sc.HasTraceID() {
        return l.With().
            Str("trace_id", sc.TraceID().String()).
            Str("span_id", sc.SpanID().String()).
            Logger()
    }
    return *l
}
```

**Cadeia de fallback:**
1. Logger do context (`zerolog.Ctx(ctx)`)
2. DefaultContextLogger (registrado no main)
3. `zerolog.Nop()` (silencioso, evita panic)

**Resultado:** Todo log emitido via `logger.Ctx(ctx)` carrega trace_id/span_id, permitindo buscar logs de uma request especifica no Loki/ELK filtrando por trace.

---

## Padrao de Uso por Camada

### Use Case (Application Layer)

Dois logs por operacao: **entrada** e **saida**. Campos estruturados com identificadores de negocio.

```go
func (uc *UseCase) Execute(ctx context.Context, cmd Command) (*Result, error) {
    log := logger.Ctx(ctx)
    log.Info().
        Str("entity_id", cmd.EntityID).
        Str("operation_type", cmd.Type).
        Str("idempotency_key", cmd.IdempotencyKey).
        Msg("executing operation")

    // ... processamento ...

    log.Info().
        Str("entity_id", result.ID).
        Str("status", result.Status).
        Msg("operation completed")
    return result, nil
}
```

### Middleware gRPC (Interceptor)

Log automatico de request/response com duracao e status gRPC:

```go
func TelemetryInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        start := time.Now()
        log := logger.Ctx(ctx)

        // Log de entrada com campos extraidos do proto
        log.Info().
            Str("rpc_method", info.FullMethod).
            Str("merchant_id", extractField(req, "merchant_id")).
            Msg("grpc request received")

        resp, err := handler(ctx, req)
        elapsed := time.Since(start)

        if err != nil {
            st, _ := grpcStatus.FromError(err)
            log.Error().Err(err).
                Str("rpc_method", info.FullMethod).
                Str("grpc_status", st.Code().String()).
                Int64("duration_ms", elapsed.Milliseconds()).
                Msg("grpc request failed")
        } else {
            log.Info().
                Str("rpc_method", info.FullMethod).
                Int64("duration_ms", elapsed.Milliseconds()).
                Msg("grpc request completed")
        }
        return resp, err
    }
}
```

### Middleware HTTP (Fiber)

Mesmo padrao adaptado para HTTP com status code e rota:

```go
func TelemetryMiddleware() fiber.Handler {
    return func(c fiber.Ctx) error {
        start := time.Now()
        // ... setup de span e context ...

        err := c.Next()
        elapsed := time.Since(start)
        statusCode := c.Response().StatusCode()
        log := logger.Ctx(ctx)

        if err != nil || statusCode >= 500 {
            log.Error().Err(err).
                Str("http_method", c.Method()).
                Str("http_route", c.Route().Path).
                Int("http_status_code", statusCode).
                Int64("duration_ms", elapsed.Milliseconds()).
                Msg("http request failed")
        } else {
            log.Info().
                Str("http_method", c.Method()).
                Str("http_route", c.Route().Path).
                Int("http_status_code", statusCode).
                Int64("duration_ms", elapsed.Milliseconds()).
                Msg("http request completed")
        }
        return err
    }
}
```

### Provider / Cliente HTTP Externo

Log de chamadas a APIs externas com duracao e status:

```go
log := logger.Ctx(ctx)
log.Info().
    Str("provider_name", "matera").
    Str("operation", "Authorize").
    Str("entity_id", pay.ID).
    Msg("calling provider")

// ... chamada HTTP ...

if err != nil {
    log.Error().Err(err).
        Str("entity_id", pay.ID).
        Int64("elapsed_ms", elapsed).
        Msg("provider call failed")
} else {
    log.Info().
        Str("entity_id", pay.ID).
        Str("provider_tx_id", resp.TransactionID).
        Int64("elapsed_ms", elapsed).
        Msg("provider call completed")
}
```

### Consumer de Mensagens (SQS/Memory)

Consumers de infraestrutura podem usar `log/slog` quando operam fora do contexto de negocio (startup, polling loop):

```go
slog.Info("consumer started", "queue_url", c.queueURL)
slog.Error("failed to receive messages", "error", err, "queue_url", c.queueURL)
```

Para handlers de mensagem, SEMPRE use `logger.Ctx(ctx)` pois o handler opera dentro de um span.

---

## Campos Estruturados: Convencoes

### Nomenclatura de Campos

| Categoria | Campos | Tipo |
|-----------|--------|------|
| Identificadores de negocio | `entity_id`, `merchant_id`, `workspace_id`, `order_id` | String |
| Rastreamento automatico | `trace_id`, `span_id` (injetados por Ctx) | String |
| Protocolo HTTP | `http_method`, `http_route`, `http_status_code` | String/Int |
| Protocolo gRPC | `rpc_method`, `grpc_status` | String |
| Performance | `duration_ms`, `elapsed_ms` | Int64 |
| Operacao | `status`, `method_type`, `event_type` | String |
| Erro | campo `.Err(err)` do zerolog | Error |
| Provider | `provider_name`, `provider_tx_id` | String |
| Messaging | `queue_url`, `retry_count`, `batch_size` | String/Int |

### Regras de Campos

1. **snake_case** para nomes de campos — consistente com JSON
2. **Duracao sempre em milissegundos** (`duration_ms`, `elapsed_ms`) — unidade explicita no nome
3. **IDs sempre como String** — mesmo que sejam UUIDs
4. **Nunca interpole strings** — use `.Str()`, `.Int()`, `.Bool()` do builder
5. **Nunca use `fmt.Sprintf` em mensagens** — structured logging apenas

---

## O Que Logar vs O Que NUNCA Logar

| Logar (seguro) | NUNCA logar (PII/segredo) |
|----------------|---------------------------|
| IDs de entidade (payment_id, order_id) | Numero do cartao (PAN) |
| IDs de merchant/workspace | CVV/CVC |
| Status e tipo de operacao | CPF/CNPJ completo |
| Codigos de erro (error_code) | Senhas e hashes de senha |
| Tipo de metodo (card, pix, boleto) | API keys e secrets |
| Chave de idempotencia | Tokens de provider |
| Duracao em ms | Bearer/JWT tokens |
| Provider transaction ID | Chaves PIX completas |
| Retry count | Dados bancarios (agencia/conta) |

**Principio:** Logue identificadores, nunca logue segredos ou dados que identifiquem pessoa fisica.

---

## Niveis de Log

| Nivel | Quando Usar | Exemplo |
|-------|-------------|---------|
| **Debug** | Troubleshooting temporario, NUNCA em producao permanente | Conteudo de request para debug |
| **Info** | Operacoes normais, entrada/saida de use case, marcos | "payment created", "grpc request completed" |
| **Warn** | Situacao inesperada mas recuperavel | "retry attempt 3 of 5", "fallback provider used" |
| **Error** | Falha que impacta a operacao | "failed to save payment", "provider timeout" |
| **Fatal/Panic** | NUNCA usar — prefira Error + return | — |

**Regra de ouro:** Em producao, nivel minimo Info. Debug level apenas temporariamente e removido antes de merge.

---

## Anti-Patterns

```go
// ERRADO: logger avulso sem context
log := zerolog.New(os.Stdout)
log.Info().Msg("something happened")  // Sem trace_id, sem correlacao

// CORRETO: logger do context
log := logger.Ctx(ctx)
log.Info().Msg("something happened")  // Com trace_id + span_id

// ERRADO: string interpolation
log.Info().Msg(fmt.Sprintf("payment %s created with status %s", id, status))

// CORRETO: campos estruturados
log.Info().Str("payment_id", id).Str("status", status).Msg("payment created")

// ERRADO: logar dentro de loop
for _, item := range items {
    log.Info().Str("item_id", item.ID).Msg("processing item")  // N logs
}

// CORRETO: logar resumo
log.Info().Int("count", len(items)).Msg("processing batch")

// ERRADO: Error sem .Err(err)
log.Error().Str("error", err.Error()).Msg("failed")

// CORRETO: Error com .Err(err)
log.Error().Err(err).Msg("failed to save payment")
```

---

## Regras

1. SEMPRE use `logger.Ctx(ctx)` — nunca crie logger avulso
2. SEMPRE structured logging (`.Str()`, `.Int()`, `.Bool()`) — nunca `fmt.Sprintf` em msgs
3. NUNCA logue dados sensiveis (PAN, CVV, CPF, tokens, secrets)
4. Dois logs por use case: entrada (Info) e saida (Info/Error)
5. Error log SEMPRE com `.Err(err)` para stack trace
6. Middleware loga request/response automaticamente — nao duplique
7. Duracoes sempre em milissegundos com sufixo `_ms` no nome do campo
8. Logger DEVE ser registrado como `DefaultContextLogger` no main
9. Correlacao com OTel e automatica via `Ctx()` — nao injete trace_id manualmente
10. Consumer de infra (startup/polling) pode usar `slog`; handlers de mensagem usam `logger.Ctx(ctx)`
