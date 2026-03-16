# Composition Root (Bootstrap)

## Quando Ativar
- Criar novo service container (ServiceCore, ServiceWebhook, ServiceSaga)
- Adicionar novo use case ou dependencia
- Configurar wiring de DI

---

## Como Pensar

O bootstrap e o unico lugar do projeto que "sabe tudo". Ele importa adapters, application e ports para conectar implementacoes concretas aos contratos.

Nao usamos framework de DI (Wire, dig). A injecao e manual e explicita. Isso significa:
- Se um use case precisa de um novo repositorio, voce adiciona o parametro no construtor E no bootstrap
- Se esqueceu de passar, o compilador reclama — zero surpresas em runtime

Pensamento: "O bootstrap e como a planta eletrica de um predio — liga os fios. Nenhum apartamento (use case) sabe de onde vem a energia (adapter)."

---

## Codigo Real: ServiceCore

```go
// internal/bootstrap/service_core.go
package bootstrap

type ServiceCore struct {
    PaymentRepo  outbound.PaymentRepository
    CompanyRepo  outbound.CompanyRepository
    CustomerRepo outbound.CustomerRepository

    CreatePaymentUC    *createpayment.UseCase
    AuthorizePaymentUC *authorizepayment.UseCase
    CapturePaymentUC   *capturepayment.UseCase
    CancelPaymentUC    *cancelpayment.UseCase
    RefundPaymentUC    *refundpayment.UseCase
}

func NewServiceCore(ctx context.Context, cfg *config.Config, log zerolog.Logger) (*ServiceCore, error) {
    db, err := NewDatabase(cfg.Database)
    if err != nil {
        return nil, err
    }

    // Adapters: repositorios
    paymentRepo := pgRepo.NewPostgresPaymentRepository(db)
    companyRepo := pgRepo.NewPostgresCompanyRepository(db)
    customerRepo := pgRepo.NewPostgresCustomerRepository(db)
    fraudRepo := pgRepo.NewPostgresFraudRepository(db)
    txnRepo := pgRepo.NewPostgresTransactionRepository(db)
    refundRepo := pgRepo.NewPostgresRefundRepository(db)
    idempotencyStore := pgRepo.NewPostgresIdempotencyStore(db)
    eventPub := noop.NewPublisher()

    // Adapters: providers
    bundle := BuildProviders(cfg, log)

    // Application: use cases (wiring explicito)
    createPaymentUC := createpayment.NewUseCase(paymentRepo, fraudRepo, bundle.Provider, eventPub, idempotencyStore)
    authorizePaymentUC := authorizepayment.NewUseCase(paymentRepo, txnRepo, bundle.Provider, eventPub)
    capturePaymentUC := capturepayment.NewUseCase(paymentRepo, txnRepo, bundle.Provider, eventPub)
    cancelPaymentUC := cancelpayment.NewUseCase(paymentRepo, txnRepo, bundle.Provider, eventPub)
    refundPaymentUC := refundpayment.NewUseCase(paymentRepo, refundRepo, txnRepo, bundle.Provider, eventPub)

    return &ServiceCore{
        PaymentRepo:        paymentRepo,
        CompanyRepo:        companyRepo,
        CustomerRepo:       customerRepo,
        CreatePaymentUC:    createPaymentUC,
        AuthorizePaymentUC: authorizePaymentUC,
        CapturePaymentUC:   capturePaymentUC,
        CancelPaymentUC:    cancelPaymentUC,
        RefundPaymentUC:    refundPaymentUC,
    }, nil
}
```

---

## Padrao de Import Aliases no Bootstrap

```go
import (
    pgRepo "github.com/.../adapters/repository/postgres"
    authorizepayment "github.com/.../application/authorize_payment"
    cancelpayment "github.com/.../application/cancel_payment"
    capturepayment "github.com/.../application/capture_payment"
    createpayment "github.com/.../application/create_payment"
    refundpayment "github.com/.../application/refund_payment"
)
```

Alias `pgRepo` para o pacote postgres evita ambiguidade. Use cases usam o package name junto (diretorio `create_payment` → import alias `createpayment`).

---

## Multi-Service Architecture

| Service | Bootstrap | Protocolo | Responsabilidade |
|---|---|---|---|
| Core | `NewServiceCore()` | gRPC | API de pagamentos |
| Webhook | `NewServiceWebhook()` | HTTP (Fiber) | Notificacoes de providers |
| Saga | `NewServiceSaga()` | SQS Consumer | Validacao de pagamentos |

Cada service tem seu proprio container. Compartilham `NewDatabase()` e `BuildProviders()`.

---

## Trade-off: DI Manual vs Framework

| Aspecto | DI Manual (nosso) | Framework (Wire/dig) |
|---|---|---|
| Erros de wiring | Compile-time | Runtime |
| Debugging | Trivial (leia o codigo) | Stack traces obscuros |
| Curva de aprendizado | Zero | Precisa aprender DSL |
| Boilerplate | Mais codigo | Menos codigo |
| Seguranca de tipos | Total | Parcial |

Escolhemos DI manual porque: erros de compile-time > erros de runtime em sistema de pagamentos.

---

## Regras

1. Bootstrap e o UNICO lugar que importa adapters
2. Use cases recebem ports (interfaces), NUNCA implementacoes concretas
3. Todo use case e construido com `NewUseCase(...)` passando dependencias
4. Cada service (core, webhook, saga) tem seu proprio container
5. NAO use framework de DI — wiring manual e explícito
6. Se adicionar nova dependencia a um use case, atualize o bootstrap
