# Tracing Distribuido & Metrics

## Quando Ativar
- Adicionar tracing em use cases, adapters ou providers
- Criar, modificar ou revisar metricas
- Configurar ou inicializar OpenTelemetry SDK
- Instrumentar novo componente (handler, repository, client HTTP)
- Configurar OTEL Collector, exporters ou sampling
- Criar ou modificar dashboards de observabilidade

---

## Como Pensar

Tracing responde **"qual caminho essa requisicao percorreu e onde gastou tempo?"**
Metrics responde **"quantas vezes X aconteceu e quanto tempo levou, em agregado?"**

Ambos usam OpenTelemetry como padrao unico. Nao use Prometheus client direto, nao use Jaeger SDK direto. OTel exporta para qualquer backend via OTLP.

### Decision Tree: Preciso de um Span Aqui?

```
Operacao cruza limite de processo/rede?
├── SIM (HTTP, gRPC, DB, fila) → Span obrigatorio, SpanKind adequado
└── NAO (logica interna)
    ├── E um use case (ponto de entrada de negocio)? → Span obrigatorio
    ├── E um marco importante dentro do use case? → span.AddEvent()
    ├── E um loop ou operacao trivial? → NAO criar span
    └── E uma funcao helper pura? → NAO criar span
```

### Decision Tree: Preciso de uma Metrica Aqui?

```
Quero responder "quantos?" ou "quanto tempo?" em agregado?
├── SIM
│   ├── Contagem de eventos → Counter (Int64Counter)
│   ├── Duracao de operacao → Histogram (Float64Histogram)
│   ├── Valor atual (conexoes ativas, fila) → Gauge (UpDownCounter)
│   └── Taxa de erro → Counter de erros / Counter total
└── NAO → Span attributes sao suficientes
```

### Trade-offs

| Decisao | Pro | Contra |
|---------|-----|--------|
| Span por use case | Visibilidade E2E de cada operacao | Overhead minimo (microsegundos) |
| Span por query DB | Identifica queries lentas no trace | Volume de spans alto |
| AddEvent vs sub-span | Leve, sem overhead de span | Nao aparece como no de arvore no Jaeger |
| Histogram com buckets explicitos | Controle sobre resolucao de percentis | Precisa definir buckets adequados ao dominio |
| Sampler ParentBased | Respeita decisao do caller | Depende de propagacao correta |
| Sampler RatioBased | Controla volume em producao | Perde traces raros (erros) |
| OTLP gRPC | Eficiente, streaming | Requer collector |
| Singleton de metrics | Acesso global simples | Acoplamento a pacote telemetry |

---

## Stack de Observabilidade

### Componentes

```
Aplicacao Go
  ├── OpenTelemetry SDK (traces + metrics)
  ├── OTLP gRPC Exporter → OTEL Collector (porta 4317)
  └── Runtime Metrics (goroutines, heap, GC)

OTEL Collector
  ├── Receivers: OTLP gRPC (4317), OTLP HTTP (4318)
  ├── Processors: Batch (timeout 5s, batch_size 512)
  └── Exporters:
      ├── Traces → Jaeger (OTLP gRPC, porta 4317)
      ├── Metrics → Prometheus (scrape, porta 8889)
      └── Debug → stdout (desenvolvimento)

Backends
  ├── Jaeger UI (porta 16686) — visualizacao de traces
  ├── Prometheus (porta 9090) — armazenamento de metricas
  └── Grafana (porta 3000) — dashboards unificados
```

### Variaveis de Ambiente

| Variavel | Default | Descricao |
|----------|---------|-----------|
| `OTEL_COLLECTOR_ENDPOINT` | `localhost:4317` | Endpoint gRPC do collector |
| `OTEL_INSECURE` | `true` | Desabilita TLS (dev only) |
| `OTEL_SAMPLE_RATIO` | `1.0` | Taxa de amostragem (0.0 a 1.0) |

---

## Inicializacao do SDK

### Bootstrap Centralizado

```go
// pkg/telemetry/telemetry.go

func Init(ctx context.Context, cfg Config) (func(context.Context) error, error) {
    res, err := setupResource(cfg)       // Resource attributes (service.name, etc.)
    conn, err := setupGRPCConn(cfg)      // Conexao gRPC com collector
    tp, err := setupTracer(ctx, conn, res, cfg) // TracerProvider
    mp, err := setupMeter(ctx, conn, res)       // MeterProvider

    // Propagadores W3C
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},  // W3C Trace Context
        propagation.Baggage{},       // W3C Baggage
    ))

    // Metricas de runtime Go (goroutines, heap, GC)
    runtime.Start(runtime.WithMinimumReadMemStatsInterval(time.Second))

    // Inicializa instrumentos de metricas
    InitMetrics()

    // Retorna funcao de shutdown
    return func(ctx context.Context) error {
        var errs []error
        errs = append(errs, tp.Shutdown(ctx))
        errs = append(errs, mp.Shutdown(ctx))
        return errors.Join(errs...)
    }, nil
}
```

### Resource Attributes (semconv)

```go
func setupResource(cfg Config) (*resource.Resource, error) {
    return resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(cfg.ServiceName),
            semconv.ServiceVersion(cfg.ServiceVersion),
            semconv.DeploymentEnvironment(cfg.Environment),
        ),
        resource.WithProcessRuntimeName(),
        resource.WithHost(),
    )
}
```

**Obrigatorios:** `service.name`, `service.version`, `deployment.environment`.
Estes aparecem em todo trace e metrica, permitindo filtrar por servico/ambiente.

### Sampler

```go
sampler := trace.ParentBased(trace.TraceIDRatioBased(cfg.SampleRatio))
```

- `ParentBased`: Respeita decisao do caller (se o parent foi amostrado, o child tambem e)
- `TraceIDRatioBased(1.0)`: 100% em dev, reduzir para 0.1-0.5 em producao conforme volume
- Em producao, considerar `AlwaysOn` para erros via tail-based sampling no collector

### Tracer Provider

```go
exporter, _ := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
tp := trace.NewTracerProvider(
    trace.WithBatcher(exporter),   // Batch async (nao bloqueia request)
    trace.WithResource(res),
    trace.WithSampler(sampler),
)
otel.SetTracerProvider(tp)
```

### Meter Provider

```go
exporter, _ := otlpmetricgrpc.New(ctx, otlpmetricgrpc.WithGRPCConn(conn))
mp := metric.NewMeterProvider(
    metric.WithReader(metric.NewPeriodicReader(exporter, metric.WithInterval(15*time.Second))),
    metric.WithResource(res),
)
otel.SetMeterProvider(mp)
```

**Intervalo de 15s:** Metricas sao exportadas a cada 15 segundos. Adequado para producao — nao afeta latencia de requests.

### Graceful Shutdown

```go
// cmd/main.go
otelShutdown, err := telemetry.Init(ctx, cfg)
defer func() {
    if err := otelShutdown(ctx); err != nil {
        log.Error().Err(err).Msg("failed to shutdown telemetry")
    }
}()

// Em signal handler:
go func() {
    <-sigCh
    server.GracefulStop()
    otelShutdown(ctx)  // Flush pendente antes de encerrar
    cancel()
}()
```

**Obrigatorio:** Sempre chamar shutdown para flush de spans/metricas pendentes antes de encerrar o processo.

---

## Tracing por Camada

### Convenção de Tracers e Spans

| Camada | Tracer Name | Span Name | SpanKind |
|--------|-------------|-----------|----------|
| Use Case | `"usecase"` | `"CreatePayment"` (PascalCase) | Internal (default) |
| Handler de Mensagem | `"handler"` | `"ValidationHandler.Handle"` | Internal |
| Repository | `"repository"` | `"INSERT payments"` (OPERATION table) | Client |
| Provider | `"provider.{name}"` | `"matera.Authorize"` (provider.Method) | Client |
| Middleware HTTP | `"webhook"` | `"POST /webhooks/matera"` (METHOD route) | Server |
| gRPC | `otelgrpc` (auto) | Auto-gerado pelo interceptor | Server |

### Use Case (Application Layer)

```go
func (uc *UseCase) Execute(ctx context.Context, cmd Command) (*Result, error) {
    start := time.Now()
    ctx, span := otel.Tracer("usecase").Start(ctx, "CreatePayment",
        trace.WithAttributes(
            attribute.String("merchant.id", cmd.MerchantID),
            attribute.String("workspace.id", cmd.WorkspaceID),
            attribute.String("idempotency.key", cmd.IdempotencyKey),
            attribute.String("payment.method_type", cmd.MethodType),
        ),
    )
    defer span.End()

    // Marcos internos via AddEvent (leve, sem sub-span)
    span.AddEvent("idempotency_hit")
    span.AddEvent("fraud_assessment_completed", trace.WithAttributes(
        attribute.Bool("fraud.blocked", assessment.ShouldBlock()),
    ))
    span.AddEvent("payment_persisted")

    // Atributos finais (resultado)
    span.SetAttributes(
        attribute.String("payment.id", p.ID),
        attribute.String("payment.status", string(p.Status)),
    )

    // Metricas
    metrics := telemetry.GetMetrics()
    if metrics != nil {
        metrics.RecordPaymentRequest(ctx, "CreatePayment", string(p.Status), time.Since(start))
    }
    return result, nil
}
```

### Repository (Database)

Helper dedicado para spans de DB com atributos semanticos:

```go
// internal/adapters/repository/postgres/tracing.go

type dbSpan struct {
    span      trace.Span
    start     time.Time
    operation string
    table     string
}

func startDBSpan(ctx context.Context, operation, table string) (context.Context, *dbSpan) {
    ctx, span := otel.Tracer("repository").Start(ctx, operation+" "+table,
        trace.WithSpanKind(trace.SpanKindClient),
    )
    span.SetAttributes(
        attribute.String("db.system", "postgresql"),
        attribute.String("db.operation", operation),
        attribute.String("db.sql.table", table),
    )
    return ctx, &dbSpan{span: span, start: time.Now(), operation: operation, table: table}
}

func endDBSpan(ds *dbSpan, err error, rowsAffected int64) {
    if rowsAffected > 0 {
        ds.span.SetAttributes(attribute.Int64("db.rows_affected", rowsAffected))
    }
    if err != nil {
        ds.span.RecordError(err)
        ds.span.SetStatus(codes.Error, err.Error())
    }
    // Metrica de DB
    if metrics := telemetry.GetMetrics(); metrics != nil {
        metrics.RecordDBQuery(ctx, ds.operation, ds.table, time.Since(ds.start))
    }
    ds.span.End()
}

// Uso em todo metodo de repository:
func (r *Repository) Save(ctx context.Context, entity *Entity) error {
    ctx, span := startDBSpan(ctx, "INSERT", "table_name")
    var err error
    defer func() { endDBSpan(span, err, 1) }()
    // ... operacao ...
    return err
}
```

### Provider / Cliente HTTP Externo

```go
// Tracer nomeado por provider (variavel de pacote)
var tracer = otel.Tracer("provider.matera")

func (p *Provider) Authorize(ctx context.Context, pay *Payment) (*Response, error) {
    ctx, span := tracer.Start(ctx, "matera.Authorize")
    defer span.End()

    // ... chamada HTTP com otelhttp.NewTransport() ...

    span.AddEvent("provider_authorized", trace.WithAttributes(
        attribute.String("provider.tx_id", resp.TransactionID),
        attribute.String("provider.status", resp.Status),
    ))
    return resp, nil
}
```

**HTTP Client com auto-instrumentacao:**

```go
httpClient := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}
```

`otelhttp.NewTransport()` cria spans automaticos para cada request HTTP, com atributos `http.method`, `http.url`, `http.status_code`.

### gRPC (Interceptor + Auto-instrumentacao)

```go
// Dual: auto-instrumentacao + enrichment customizado
grpcServer := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),     // Spans automaticos
    grpc.ChainUnaryInterceptor(
        grpcAdapter.TelemetryInterceptor(),              // Enrichment customizado
    ),
)
```

**Extracao automatica de campos Proto:**

```go
func extractProtoFields(req any) map[string]string {
    fieldMap := map[string]string{
        "merchant_id":     "merchant.id",
        "workspace_id":    "workspace.id",
        "payment_id":      "payment.id",
        "idempotency_key": "idempotency.key",
    }
    // Usa ProtoReflect para extrair valores string do request
}
```

### Middleware HTTP (Fiber)

```go
func TelemetryMiddleware() fiber.Handler {
    tracer := otel.Tracer("webhook")
    propagator := otel.GetTextMapPropagator()

    return func(c fiber.Ctx) error {
        // Extrai trace context do header (W3C traceparent)
        carrier := &fiberHeaderCarrier{ctx: c}
        parentCtx := propagator.Extract(c.Context(), carrier)

        spanName := fmt.Sprintf("%s %s", c.Method(), c.Route().Path)
        ctx, span := tracer.Start(parentCtx, spanName,
            trace.WithSpanKind(trace.SpanKindServer),
            trace.WithAttributes(
                attribute.String("http.method", c.Method()),
                attribute.String("http.route", c.Route().Path),
            ),
        )
        defer span.End()

        // Injeta trace_id no response header
        sc := span.SpanContext()
        if sc.HasTraceID() {
            c.Set("X-Trace-Id", sc.TraceID().String())
        }

        c.SetContext(ctx)
        // ... c.Next() + error handling ...
    }
}
```

**Carrier customizado para Fiber** (necessario porque Fiber nao usa `http.Header`):

```go
type fiberHeaderCarrier struct {
    ctx fiber.Ctx
}
func (c *fiberHeaderCarrier) Get(key string) string    { return c.ctx.Get(key) }
func (c *fiberHeaderCarrier) Set(key, value string)     { c.ctx.Set(key, value) }
func (c *fiberHeaderCarrier) Keys() []string            { /* VisitAll */ }
```

---

## Propagacao de Trace ID

O trace_id flui por tres canais:

1. **Context Go** — `context.Context` carrega o span ativo; toda funcao que recebe `ctx` participa do trace
2. **Headers HTTP** — W3C `traceparent` header, extraido/injetado pelo propagador
3. **Response Headers** — `X-Trace-Id` devolvido ao cliente para debugging

```
Cliente → [traceparent header] → Middleware HTTP/gRPC
  → Extrai parent context
  → Cria span filho
  → Injeta X-Trace-Id no response
  → Passa ctx para use case → repository → provider
  → Cada camada cria sub-span com mesmo trace_id
```

---

## Tratamento de Erros em Spans

**SEMPRE use RecordError + SetStatus juntos:**

```go
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, "descricao legivel do erro")
    return nil, err
}
```

- `RecordError` grava o erro como evento no span (visivel no Jaeger como exception event)
- `SetStatus(codes.Error, ...)` marca o span como falho (aparece vermelho no Jaeger)
- **Descricao do SetStatus deve ser legivel**, nao o err.Error() cru (ex: "failed to save payment", nao "pq: unique_violation")

---

## Metricas

### Singleton Global

```go
// pkg/telemetry/metrics.go

var (
    globalMetrics *Metrics
    metricsOnce   sync.Once
)

func InitMetrics() (*Metrics, error) {
    var initErr error
    metricsOnce.Do(func() {
        meter := otel.Meter("service-name")
        globalMetrics = &Metrics{}
        // ... registra instrumentos ...
    })
    return globalMetrics, initErr
}

func GetMetrics() *Metrics {
    if globalMetrics == nil {
        m, _ := InitMetrics()
        return m
    }
    return globalMetrics
}
```

**`sync.Once`** garante inicializacao thread-safe. `GetMetrics()` e nil-safe — retorna instancia ou inicializa sob demanda.

### Instrumentos de Metricas

#### Counters (Int64Counter)

```go
meter := otel.Meter("service-name")

requestsTotal, _ := meter.Int64Counter("payment_requests_total",
    metric.WithDescription("Total number of payment requests"),
)

errorsTotal, _ := meter.Int64Counter("payment_errors_total",
    metric.WithDescription("Total number of payment errors"),
)

providerRequestsTotal, _ := meter.Int64Counter("provider_requests_total",
    metric.WithDescription("Total number of provider requests"),
)

dbQueriesTotal, _ := meter.Int64Counter("db_queries_total",
    metric.WithDescription("Total number of database queries"),
)

webhookEventsTotal, _ := meter.Int64Counter("webhook_events_total",
    metric.WithDescription("Total number of webhook events"),
)
```

#### Histograms (Float64Histogram)

```go
var durationBuckets = metric.WithExplicitBucketBoundaries(
    5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000,
)

requestsDuration, _ := meter.Float64Histogram("payment_requests_duration",
    metric.WithDescription("Payment request duration in milliseconds"),
    metric.WithUnit("ms"),
    durationBuckets,
)

providerDuration, _ := meter.Float64Histogram("provider_requests_duration",
    metric.WithDescription("Provider request duration in milliseconds"),
    metric.WithUnit("ms"),
    durationBuckets,
)

dbDuration, _ := meter.Float64Histogram("db_queries_duration",
    metric.WithDescription("Database query duration in milliseconds"),
    metric.WithUnit("ms"),
    durationBuckets,
)
```

**Buckets explícitos:** 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000 ms. Cobre desde queries rapidas (5ms) ate operacoes lentas (10s). Ajustar conforme SLO do servico.

### Metodos de Gravacao

```go
// Counter: Add(ctx, 1, attributes)
func (m *Metrics) RecordPaymentRequest(ctx context.Context, method, status string, duration time.Duration) {
    attrs := metric.WithAttributes(
        attribute.String("method", method),
        attribute.String("status", status),
    )
    m.PaymentRequestsTotal.Add(ctx, 1, attrs)
    m.PaymentRequestsDuration.Record(ctx, float64(duration.Milliseconds()), attrs)
}

func (m *Metrics) RecordPaymentError(ctx context.Context, method, errorType string) {
    m.PaymentErrorsTotal.Add(ctx, 1, metric.WithAttributes(
        attribute.String("method", method),
        attribute.String("error_type", errorType),
    ))
}

func (m *Metrics) RecordProviderRequest(ctx context.Context, provider, operation string, statusCode int, duration time.Duration) {
    attrs := metric.WithAttributes(
        attribute.String("provider_name", provider),
        attribute.String("operation", operation),
        attribute.Int("status_code", statusCode),
    )
    m.ProviderRequestsTotal.Add(ctx, 1, attrs)
    m.ProviderRequestsDuration.Record(ctx, float64(duration.Milliseconds()), attrs)
}

func (m *Metrics) RecordDBQuery(ctx context.Context, operation, table string, duration time.Duration) {
    attrs := metric.WithAttributes(
        attribute.String("operation", operation),
        attribute.String("table", table),
    )
    m.DBQueriesTotal.Add(ctx, 1, attrs)
    m.DBQueriesDuration.Record(ctx, float64(duration.Milliseconds()), attrs)
}

func (m *Metrics) RecordWebhookEvent(ctx context.Context, provider, eventType string, processed bool) {
    m.WebhookEventsTotal.Add(ctx, 1, metric.WithAttributes(
        attribute.String("provider", provider),
        attribute.String("event_type", eventType),
        attribute.Bool("processed", processed),
    ))
}
```

### Metricas de Dominio Especifico (Saga)

Para fluxos especificos (saga, batch, pipeline), criar struct de metricas separada:

```go
type SagaMetrics struct {
    ValidationReceived  metric.Int64Counter
    ValidationSucceeded metric.Int64Counter
    ValidationFailed    metric.Int64Counter
    ValidationError     metric.Int64Counter
    ValidationDuration  metric.Float64Histogram
}
```

Mesmo padrao: `sync.Once` + `GetSagaMetrics()` global.

### Separacao: Business vs Technical

| Tipo | Metricas | Quem Grava |
|------|----------|------------|
| **Business** | payment_requests_total, payment_errors_total, webhook_events_total, saga.validation.* | Use cases |
| **Technical** | provider_requests_total/duration, db_queries_total/duration | Adapters (repository, provider) |
| **Runtime** | goroutines, heap_alloc, gc_pause | Automatico (runtime.Start) |

Business metrics refletem KPIs de negocio. Technical metrics refletem saude da infraestrutura. Runtime metrics sao coletadas automaticamente.

---

## Convencoes de Atributos (Span e Metrics)

### Naming: dot.notation

```
merchant.id          (nao merchantId, nao merchant_id)
payment.status       (nao paymentStatus)
payment.method_type  (nao methodType)
db.system            (nao dbSystem)
db.operation         (nao dbOperation)
http.method          (nao httpMethod)
provider.name        (nao providerName)
provider.tx_id       (nao providerTxId)
```

Segue convencao OpenTelemetry Semantic Conventions (semconv). Usar dot notation para hierarquia.

### Atributos Comuns por Camada

| Camada | Atributos |
|--------|-----------|
| Use Case | `merchant.id`, `workspace.id`, `idempotency.key`, `payment.method_type`, `payment.id`, `payment.status` |
| Repository | `db.system`, `db.operation`, `db.sql.table`, `db.rows_affected` |
| Provider | `provider.name`, `provider.tx_id`, `http.method`, `http.url`, `http.status_code` |
| Middleware HTTP | `http.method`, `http.route`, `http.status_code`, `duration_ms` |
| Middleware gRPC | `rpc.method`, `rpc.grpc.status_code`, `duration_ms` |
| Handler | `message.size_bytes`, `order.id` |

---

## OTEL Collector Config

```yaml
# deployments/otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 512

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889
  debug:
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger, debug]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus, debug]
```

**Batch processor:** Agrupa spans/metricas antes de exportar. Reduz overhead de rede. Timeout 5s + batch_size 512 e um bom default.

---

## Dashboard (Grafana)

### Panels Essenciais

| Secao | Panel | Query PromQL |
|-------|-------|--------------|
| Overview | Request Rate | `sum(rate(payment_requests_total[5m]))` |
| Overview | Error Rate % | `sum(rate(payment_errors_total[5m])) / clamp_min(sum(rate(payment_requests_total[5m])), 1) * 100` |
| Overview | P99 Latency | `histogram_quantile(0.99, sum(rate(payment_requests_duration_bucket[5m])) by (le))` |
| Overview | P50 Latency | `histogram_quantile(0.50, sum(rate(payment_requests_duration_bucket[5m])) by (le))` |
| Payments | Rate by Method | `sum(rate(payment_requests_total[5m])) by (method)` |
| Payments | Errors by Type | `sum(rate(payment_errors_total[5m])) by (method, error_type)` |
| Payments | Latency P50/P90/P99 | `histogram_quantile(0.{50,90,99}, ...)` |
| Provider | Request Rate | `sum(rate(provider_requests_total[5m])) by (provider_name, operation)` |
| Provider | Latency P99 | `histogram_quantile(0.99, ...) by (provider_name)` |
| Database | Query Rate | `sum(rate(db_queries_total[5m])) by (operation, table)` |
| Database | Query Latency P99 | `histogram_quantile(0.99, ...) by (table)` |
| Webhooks | Events by Provider | `sum(rate(webhook_events_total[5m])) by (provider, event_type)` |
| Runtime | Goroutines | `process_runtime_go_goroutines` |
| Runtime | Heap Memory | `process_runtime_go_mem_heap_alloc` |
| Runtime | GC Pause | `rate(process_runtime_go_gc_pause_ns_sum[5m]) / rate(process_runtime_go_gc_pause_ns_count[5m])` |

### Datasources

- **Prometheus** para metricas e runtime
- **Jaeger/Tempo** para traces (linkado via trace_id)

---

## Metricas de Runtime (Automaticas)

```go
import "go.opentelemetry.io/contrib/instrumentation/runtime"

runtime.Start(runtime.WithMinimumReadMemStatsInterval(time.Second))
```

Coleta automaticamente:
- `process_runtime_go_goroutines` — Goroutines ativas
- `process_runtime_go_mem_heap_alloc` — Heap alocado
- `process_runtime_go_gc_pause_ns` — Duracao de GC pause
- `process_runtime_go_mem_heap_objects` — Objetos no heap

Sem codigo adicional. Apenas `runtime.Start()` na inicializacao.

---

## Health Check de Providers

```go
type Provider struct {
    healthy atomic.Bool
}

func (p *Provider) IsHealthy(ctx context.Context) bool {
    return p.healthy.Load()
}
```

- Cada provider mantem `atomic.Bool` de saude
- Router filtra apenas providers saudaveis antes de selecionar
- Nao e um health check endpoint HTTP — e logica interna de roteamento

---

## Regras

1. OpenTelemetry para tracing E metrics — nao use Prometheus client direto, nao use Jaeger SDK direto
2. Tracer nomeado por camada: `"usecase"`, `"repository"`, `"handler"`, `"provider.{name}"`
3. `defer span.End()` SEMPRE presente — sem excecao
4. `span.RecordError(err)` + `span.SetStatus(codes.Error, ...)` SEMPRE juntos
5. Atributos em dot.notation (semconv) — sem dados sensiveis
6. `telemetry.GetMetrics()` retorna singleton nil-safe — SEMPRE check `if metrics != nil`
7. Todo metodo de repository cria span de DB via helper (`startDBSpan`/`endDBSpan`)
8. Use `span.AddEvent()` para marcos internos, NAO sub-spans
9. SpanKind: Server para endpoints, Client para DB/provider, Internal para use cases
10. Histograms com buckets explicitos adequados ao dominio (5ms-10s para web)
11. Business metrics nos use cases, technical metrics nos adapters
12. Graceful shutdown OBRIGATORIO — chame `otelShutdown(ctx)` antes de encerrar
13. HTTP clients externos com `otelhttp.NewTransport()` para auto-instrumentacao
14. Middleware HTTP/gRPC deve injetar `X-Trace-Id` no response header
15. W3C TraceContext + Baggage como propagadores — nunca propague trace_id manualmente
