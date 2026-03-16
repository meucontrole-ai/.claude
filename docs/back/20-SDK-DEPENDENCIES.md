# SDK & Dependencies

## Quando Ativar
- Adicionar nova dependencia ao projeto
- Integrar SDK de servico externo
- Avaliar seguranca de dependencias
- Configurar payment provider

---

## Como Pensar

Cada dependencia e divida tecnica potencial. Antes de adicionar, pergunte:
1. "A stdlib resolve?" — Se sim, nao adicione
2. "Quem mantem?" — ultimo commit, stars, issues abertas
3. "Tem CVEs?" — `govulncheck ./...`

Para SDKs de providers externos (Matera, Stripe), a regra e: NUNCA exponha tipos da SDK fora do adapter. O domain e application so veem interfaces (ports outbound).

---

## Dependencias do Projeto

| Categoria | Biblioteca | Uso |
|---|---|---|
| RPC Framework | gRPC + Protobuf | API principal |
| HTTP Framework | Fiber | Webhooks |
| Database ORM | GORM v2 | PostgreSQL |
| Logging | Zerolog | Structured logging |
| Tracing/Metrics | OpenTelemetry | Observabilidade |
| Testing | Testify (assert/require) | Assercoes |
| UUID | google/uuid | Geracao de IDs |
| AWS SDK | aws-sdk-go-v2 | EventBridge, SQS |
| Config | stdlib (os.Getenv) | Environment variables |

---

## Criterios para Adicionar Dependencia

1. **Necessidade real**: stdlib nao resolve?
2. **Manutencao ativa**: ultimo commit < 6 meses
3. **Seguranca**: `govulncheck ./...` sem CVEs
4. **Licenca**: MIT, Apache 2.0, BSD
5. **Tamanho**: nao infla binario desnecessariamente

---

## Provider Registry + Router

### Registry

```go
// internal/adapters/providers/registry.go
type Registry struct {
    mu        sync.RWMutex
    providers map[string]outbound.PaymentProvider
}

func (r *Registry) Register(name string, p outbound.PaymentProvider) { ... }
func (r *Registry) Get(name string) (outbound.PaymentProvider, bool) { ... }
```

### SmartRouter

Seleciona provider por prioridade configuravel e health status:

```go
// internal/adapters/providers/router.go
type SmartRouter struct {
    registry *Registry
    priority []string // ex: ["matera", "stripe", "c6"]
}

func (r *SmartRouter) SelectProvider(ctx context.Context) (outbound.PaymentProvider, error) {
    for _, name := range r.priority {
        if p, ok := r.registry.Get(name); ok && p.IsHealthy(ctx) {
            return p, nil
        }
    }
    return nil, shared.NewDomainError(shared.ErrCodeInvariantViolation, "no healthy provider")
}
```

### ResilientProvider

```go
// internal/adapters/providers/resilient_provider.go
type ResilientProvider struct {
    inner   outbound.PaymentProvider
    healthy atomic.Bool
}
```

Wrapper com retry e health tracking. Lock-free via `atomic.Bool`.

---

## PaymentProvider Interface Real

```go
// internal/ports/outbound/payment_provider.go
type PaymentProvider interface {
    Name() string
    SupportsMethod(methodType string) bool
    Authorize(ctx context.Context, p *payment.Payment) (*payment.ProviderResult, error)
    Capture(ctx context.Context, p *payment.Payment, amount shared.Money) (*payment.ProviderResult, error)
    Cancel(ctx context.Context, p *payment.Payment, reason string) (*payment.ProviderResult, error)
    Refund(ctx context.Context, p *payment.Payment, amount shared.Money, reason string) (*payment.ProviderResult, error)
    IsHealthy(ctx context.Context) bool
}
```

Implementacoes em `adapters/providers/<provider>/provider.go`.

---

## Regras para SDKs Externos

1. SEMPRE encapsule atras de interface (port outbound)
2. SEMPRE configure timeout em clients HTTP
3. SEMPRE propague context
4. SEMPRE converta erros de SDK para DomainError
5. NUNCA exponha tipos da SDK no domain ou application
6. Logue chamadas externas com latencia (sem dados sensiveis)
