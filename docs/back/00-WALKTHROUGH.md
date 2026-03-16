# Walkthrough: CreatePayment End-to-End

## Quando Ativar
- Entender o projeto do zero
- Entender o fluxo completo de uma operacao
- Servir como modelo mental para implementar novas operacoes

---

## Visao Geral

Este documento percorre o fluxo completo de `CreatePayment` — da requisicao gRPC ate a persistencia no banco e publicacao de eventos. Cada camada e mostrada com codigo real do projeto.

---

## 1. Proto Request (Cliente gRPC)

O cliente envia um `CreatePaymentRequest` via gRPC. O proto define a estrutura da mensagem.

---

## 2. gRPC Handler (Adapter de Entrada)

```go
// internal/adapters/grpc/handler/payment_handler.go
package handler

func (s *PaymentService) CreatePayment(ctx context.Context, req *paymentProto.CreatePaymentRequest) (*paymentProto.PaymentResponse, error) {
    // Proto -> Command (campo a campo, sem logica)
    cmd := payment.CreatePaymentCommand{
        MerchantID:         req.MerchantId,
        WorkspaceID:        req.WorkspaceId,
        IdempotencyKey:     req.IdempotencyKey,
        Amount:             req.Amount,
        Currency:           req.Currency,
        CaptureMethod:      req.CaptureMethod,
        ConfirmationMethod: req.ConfirmationMethod,
        MethodType:         req.MethodType,
        Description:        req.Description,
        Metadata:           req.Metadata,
    }

    if req.Card != nil {
        cmd.Card = &payment.CardInput{
            Token:           req.Card.Token,
            Brand:           req.Card.Brand,
            LastFourDigits:  req.Card.LastFourDigits,
            ExpirationMonth: int(req.Card.ExpirationMonth),
            ExpirationYear:  int(req.Card.ExpirationYear),
            HolderName:      req.Card.HolderName,
            Funding:         req.Card.Funding,
            Installments:    int(req.Card.Installments),
            BinNumber:       req.Card.BinNumber,
        }
    }
    // ... pix, boleto, billing, splits, fraud_data

    // Chama use case e converte resultado
    res, err := s.createPaymentUC.Execute(ctx, cmd)
    if err != nil {
        return nil, status.Error(codes.Internal, err.Error())
    }
    return mapper.ToPaymentResponsePB(res), nil
}
```

O handler:
- Converte tipos proto para primitivos Go (`int(req.Card.ExpirationMonth)`)
- Monta o command com campos condicionais (`if req.Card != nil`)
- Chama o use case (interface inbound)
- Converte resultado via mapper do package `mapper`

---

## 3. Use Case (Application Layer)

```go
// internal/application/create_payment/usecase.go
package createpayment

type UseCase struct {
    paymentRepo outbound.PaymentRepository
    fraudRepo   outbound.FraudAssessmentRepository
    provider    outbound.PaymentProvider
    eventPub    outbound.EventPublisher
    idempotency outbound.IdempotencyStore
}

func (uc *UseCase) Execute(ctx context.Context, cmd payment.CreatePaymentCommand) (*payment.PaymentResult, error) {
    // 1. Tracing + Logging
    start := time.Now()
    ctx, span := otel.Tracer("usecase").Start(ctx, "CreatePayment", ...)
    defer span.End()
    log := logger.Ctx(ctx)
    log.Info().Str("merchant_id", cmd.MerchantID).Msg("executing create payment")

    // 2. Idempotency check (PRIMEIRO!)
    existing, err := uc.paymentRepo.FindByIdempotencyKey(ctx, cmd.MerchantID, cmd.IdempotencyKey)
    if err == nil && existing != nil {
        return appshared.ToPaymentResult(existing), nil  // Retorna existente
    }

    // 3. Conversao primitivos -> tipos ricos
    money, err := shared.NewMoney(cmd.Amount, shared.Currency(cmd.Currency))
    if err != nil {
        return nil, shared.WrapDomainError(shared.ErrCodeInvalidArgument, "invalid amount/currency", err)
    }

    method, err := buildMethod(cmd)  // Helper privado
    if err != nil {
        return nil, err
    }

    billing := buildBilling(cmd.Billing)  // Helper privado

    // 4. Cria agregado (validacao no construtor)
    p, err := payment.NewPayment(payment.CreateParams{
        MerchantID:         cmd.MerchantID,
        WorkspaceID:        cmd.WorkspaceID,
        IdempotencyKey:     cmd.IdempotencyKey,
        Amount:             money,
        CaptureMethod:      payment.CaptureMethod(cmd.CaptureMethod),
        ConfirmationMethod: payment.ConfirmationMethod(cmd.ConfirmationMethod),
        Method:             method,
        Billing:            billing,
        Description:        cmd.Description,
        Metadata:           cmd.Metadata,
    })
    if err != nil {
        return nil, err
    }

    // 5. Fraud check (opcional)
    if cmd.FraudData != nil {
        assessment := fraud.NewAssessment(p.ID)
        assessment.SetDeviceInfo(...)
        if assessment.ShouldBlock() {
            _ = p.Fail("FRAUD_BLOCKED", "payment blocked by fraud assessment")
        }
        uc.fraudRepo.Save(ctx, assessment)
    }

    // 6. Persistencia
    if err := uc.paymentRepo.Save(ctx, p); err != nil {
        return nil, fmt.Errorf("failed to save payment: %w", err)
    }

    // 7. Publicacao de eventos (APOS persistencia!)
    events := p.PullEvents()
    if len(events) > 0 {
        uc.eventPub.Publish(ctx, events...)  // Falha nao reverte
    }

    // 8. Metrics + Log final
    metrics.RecordPaymentRequest(ctx, "CreatePayment", string(p.Status), time.Since(start))
    log.Info().Str("payment_id", p.ID).Str("status", string(p.Status)).Msg("payment created")

    // 9. Retorna resultado
    return appshared.ToPaymentResult(p), nil
}
```

Sequencia critica do use case:
1. Tracing/Logging
2. Idempotency (retorna cedo se duplicado)
3. Converte primitivos para tipos ricos (Money, Method)
4. Cria agregado (validacao no construtor)
5. Fraud check opcional
6. Salva no banco
7. Publica eventos (pos-persistencia)
8. Metrics
9. Retorna result

---

## 4. Domain Aggregate (Construtor)

```go
// internal/domain/payment/payment.go
func NewPayment(params CreateParams) (*Payment, error) {
    // Validacao fail-fast
    if params.MerchantID == "" {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "merchant_id is required")
    }
    // ... mais validacoes ...

    now := time.Now().UTC()
    p := &Payment{
        ID:     shared.GenerateID(),
        Status: StatusCreated,
        // ... todos os campos ...
        Version: 1,
    }

    // Grava evento de criacao
    p.RecordEvent(PaymentCreatedEvent{
        BaseEvent: shared.BaseEvent{
            ID:            shared.GenerateID(),
            Type:          EventPaymentCreated,
            AggregateRef:  p.ID,
            AggregateKind: "payment",
            Timestamp:     now,
        },
        MerchantID: p.MerchantID,
        Amount:     p.Amount,
    })

    return p, nil
}
```

O construtor:
- Valida invariantes (fail-fast)
- Gera ID via UUID v4 36chars
- Inicializa com `StatusCreated`, `Version: 1`
- Grava `PaymentCreatedEvent`

---

## 5. Repositorio (Adapter de Saida)

```go
// internal/adapters/repository/postgres/payment_repository.go
func (r *PostgresPaymentRepository) Save(ctx context.Context, p *payment.Payment) error {
    model := ToModel(p)  // Domain -> GORM model

    result := r.db.WithContext(ctx).Create(model)
    if result.Error != nil {
        if isDuplicateKeyError(result.Error) {
            return shared.WrapDomainError(shared.ErrCodeConflict, "payment already exists", result.Error)
        }
        return shared.WrapDomainError(shared.ErrCodeInvariantViolation, "failed to save", result.Error)
    }
    return nil
}
```

O repositorio:
- Converte domain -> model via `ToModel()`
- Usa `WithContext(ctx)` para tracing
- Detecta duplicate key -> `ErrCodeConflict`
- Nunca retorna erro GORM puro

---

## 6. Mapper Domain -> Model

```go
// internal/adapters/repository/postgres/mapper.go
func ToModel(p *payment.Payment) *PaymentModel {
    return &PaymentModel{
        ID:                 p.ID,
        MerchantID:         p.MerchantID,
        AmountCentavos:     p.Amount.Amount,      // Money -> int64
        AmountCurrency:     string(p.Amount.Currency),  // Currency -> string
        Status:             string(p.Status),      // Status -> string
        MethodType:         string(p.Method.Type), // MethodType -> string
        MethodData:         marshalMethodData(p.Method),  // Method -> JSONB
        Version:            p.Version,
        // ...
    }
}
```

Conversoes explicitas: `shared.Money` desnormaliza para duas colunas, `Method` vira JSONB.

---

## 7. Result Mapper (Domain -> DTO)

```go
// internal/application/shared/mapper.go
func ToPaymentResult(p *payment.Payment) *payment.PaymentResult {
    result := &payment.PaymentResult{
        ID:       p.ID,
        Status:   p.Status.String(),
        Amount:   p.Amount.Amount,
        Currency: string(p.Amount.Currency),
        CreatedAt: p.CreatedAt.Format("2006-01-02T15:04:05Z"),
        // ...
    }
    if p.Method.Pix != nil {
        result.PixQRCode = p.Method.Pix.QRCode
    }
    return result
}
```

Converte tipos ricos de volta para primitivos. Vive em `application/shared/` porque e compartilhado entre use cases.

---

## 8. Proto Response Mapper

```go
// internal/adapters/grpc/mapper/mapper.go
func ToPaymentResponsePB(r *payment.PaymentResult) *paymentProto.PaymentResponse {
    return &paymentProto.PaymentResponse{
        Id:       r.ID,
        Status:   r.Status,
        Amount:   r.Amount,
        // ...
    }
}
```

Ultimo passo: `PaymentResult` (primitivos) -> Proto response. Retorna ao cliente.

---

## Fluxo Resumido

```
Cliente gRPC
  -> handler/payment_handler.go (Proto -> Command)
    -> application/create_payment/usecase.go (orquestracao)
      -> domain/payment/payment.go (NewPayment: validacao + evento)
      -> adapters/repository/postgres/ (Save: domain -> model -> DB)
      -> PullEvents -> eventPub.Publish (pos-persistencia)
    <- application/shared/mapper.go (Payment -> PaymentResult)
  <- adapters/grpc/mapper/mapper.go (Result -> Proto)
Cliente gRPC
```

---

## Como Adicionar Nova Operacao (Checklist)

Para implementar uma operacao similar (ex: novo use case):

1. **Domain**: Crie o command em `domain/<aggregate>/command.go`
2. **Domain**: Crie o result em `domain/<aggregate>/result.go` (se diferente)
3. **Domain**: Adicione metodo de mutacao no agregado
4. **Domain**: Crie o evento em `domain/<aggregate>/events.go`
5. **Ports**: Crie interface inbound em `ports/inbound/<operacao>.go`
6. **Application**: Crie `application/<operacao>/usecase.go` com `UseCase` + `Execute`
7. **Adapter**: Adicione handler em `adapters/grpc/handler/`
8. **Adapter**: Adicione mapper se necessario em `adapters/grpc/mapper/`
9. **Bootstrap**: Wire o novo use case em `bootstrap/service_core.go`
10. **Teste**: Table-driven test com spy publisher

Ordem: Domain primeiro, de dentro para fora. Nunca comece pelo adapter.
