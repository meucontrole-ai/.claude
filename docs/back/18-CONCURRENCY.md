# Concurrency Patterns

## Quando Ativar
- Implementar goroutines, channels, workers
- Usar sync primitives (Mutex, RWMutex, Once, atomic)
- Resolver race conditions
- Implementar lifecycle management de goroutines

---

## Como Pensar

Go facilita concorrencia, mas nao elimina bugs. A regra e: "se dois goroutines podem acessar o mesmo dado, proteja."

Decisao rapida:
- **Ler e escrever dados compartilhados?** → Mutex/RWMutex
- **Apenas trocar flag true/false?** → atomic.Bool
- **Inicializar uma unica vez?** → sync.Once
- **Transferir ownership?** → Channel
- **Agregar resultados paralelos?** → errgroup

---

## Padroes Reais do Projeto

### Registry (RWMutex)

```go
// internal/adapters/providers/registry.go
type Registry struct {
    mu        sync.RWMutex
    providers map[string]outbound.PaymentProvider
}

func (r *Registry) Get(name string) (outbound.PaymentProvider, bool) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    p, ok := r.providers[name]
    return p, ok
}

func (r *Registry) Register(name string, p outbound.PaymentProvider) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.providers[name] = p
}
```

`RWMutex` porque leituras sao muito mais frequentes que escritas. Multiplos readers simultaneos, writer exclusivo.

### Health Check (atomic.Bool)

```go
// internal/adapters/providers/resilient_provider.go
type ResilientProvider struct {
    inner   outbound.PaymentProvider
    healthy atomic.Bool
}

func (p *ResilientProvider) SetHealthy(h bool) { p.healthy.Store(h) }
func (p *ResilientProvider) IsHealthy() bool   { return p.healthy.Load() }
```

Lock-free para status de saude — alto throughput sem contencao.

### Metrics Singleton (sync.Once)

```go
var globalMetrics *Metrics
var metricsOnce sync.Once

func InitMetrics() *Metrics {
    metricsOnce.Do(func() { globalMetrics = &Metrics{...} })
    return globalMetrics
}
```

### Spy Publisher (sync.Mutex em testes)

```go
type spyPublisher struct {
    mu     sync.Mutex
    events []saga.ValidationCompletedEvent
    err    error
}
```

Mutex no spy porque testes com `-race` detectam acessos concorrentes.

---

## Quando Usar Channel vs Mutex

| Cenario | Use |
|---|---|
| Transferir ownership de dados | Channel |
| Coordenar/sinalizar eventos | Channel |
| Proteger estado compartilhado (cache, map) | Mutex |
| Limitar concorrencia (semaphore) | Buffered channel |
| Agregar resultados paralelos | errgroup |

---

## errgroup

```go
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { orders, err = uc.orderRepo.Find(ctx, id); return err })
g.Go(func() error { stats, err = uc.statsRepo.Get(ctx, id); return err })
if err := g.Wait(); err != nil { return nil, err }
```

---

## Context Patterns

```go
ctx, cancel := context.WithTimeout(parentCtx, 30*time.Second)
defer cancel()
result, err := externalService.Call(ctx)
```

---

## Regras

1. NUNCA lance goroutine sem ownership de lifecycle
2. SEMPRE use context para cancelamento/timeout
3. SEMPRE defer para cleanup (cancel, close, unlock)
4. NUNCA compartilhe memoria sem protecao (mutex, atomic, channel)
5. `-race` flag obrigatoria nos testes: `go test ./... -race`
6. Channels fechados pelo produtor, nunca pelo consumidor
7. Recovery em goroutines background (panic recovery)
8. RWMutex quando leituras >> escritas, Mutex quando equivalentes

---

## Armadilha: Optimistic locking incompleto

Coberto em `11-REPOSITORY-PATTERN.md`, repetindo porque é crítico: se dois processos atualizam o mesmo agregado simultaneamente e seu `Updates(map)` não inclui `version` OU não checa `RowsAffected`, o último write vence silenciosamente.

```go
// ❌ errado — last-write-wins
db.Updates(map[string]interface{}{"status": m.Status})

// ✅ correto
result := db.Where("id = ? AND version = ?", m.ID, originalVersion).
    Updates(map[string]interface{}{
        "status":  m.Status,
        "version": m.Version,
    })
if result.RowsAffected == 0 {
    return shared.NewDomainError(shared.ErrCodeConflict, "stale version")
}
```

---

## Armadilha: read-after-write em serviços com múltiplas conexões

Se o frontend faz `POST /x` seguido de `PATCH /x/:id` e o backend tem pool de conexões > 1, o write do POST pode ainda não ter refletido quando o PATCH lê.

Mitigações:
- `MaxOpenConns=1` se throughput permitir
- `SELECT pg_advisory_xact_lock(hash)` dentro da transaction pra forçar ordem
- Idempotência por estado-desejado no PATCH (cliente manda o state inteiro, não diff) — elimina race por design

---

## Goroutine fire-and-forget (notify assíncrono)

Comum em publish pra outbox/webhook/broker. Use com proteção:

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Error().Any("panic", r).Msg("background task panicked")
        }
    }()

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := client.Notify(ctx, params); err != nil {
        log.Error().Err(err).Msg("notify failed")
        // considerar agendar retry via outbox
    }
}()
```

Nunca `go fn()` sem timeout — goroutines vazadas acumulam memória. Sempre `recover()` pra panic não derrubar o processo.
