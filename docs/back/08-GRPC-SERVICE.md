# gRPC Service

## Quando Ativar
- Criar ou modificar handler gRPC
- Adicionar novo RPC method
- Trabalhar com proto → command mapping
- Modificar mapper proto ↔ domain

---

## Como Pensar

O gRPC handler e um adapter fino. Ele faz APENAS:
1. Recebe proto request
2. Converte para domain command (campo a campo, sem logica)
3. Chama use case
4. Converte result para proto response via mapper

ZERO logica de negocio no handler. Se voce esta escrevendo `if` de validacao no handler, esta errado — a validacao pertence ao domain ou use case.

---

## Estrutura de Diretorios

```
internal/adapters/grpc/
  handler/                # package handler
    service.go            # PaymentService struct + construtor
    payment_handler.go    # CreatePayment, Authorize, Capture, Cancel, Refund
    company_handler.go    # RPCs de company
    customer_handler.go   # RPCs de customer
  mapper/                 # package mapper (separado!)
    mapper.go             # ToPaymentResponsePB, ToCompanyPB...
  interceptor.go          # Interceptors (logging, recovery)
```

IMPORTANTE: handler e mapper sao packages SEPARADOS (`package handler`, `package mapper`). Nao sao o mesmo package.

---

## PaymentService (Struct)

```go
// internal/adapters/grpc/handler/service.go
package handler

type PaymentService struct {
    paymentProto.UnimplementedPaymentServiceServer

    companyRepo  outbound.CompanyRepository
    customerRepo outbound.CustomerRepository

    createPaymentUC    inbound.CreatePaymentUseCase
    authorizePaymentUC inbound.AuthorizePaymentUseCase
    capturePaymentUC   inbound.CapturePaymentUseCase
    cancelPaymentUC    inbound.CancelPaymentUseCase
    refundPaymentUC    inbound.RefundPaymentUseCase
}

func NewPaymentService(
    companyRepo outbound.CompanyRepository,
    customerRepo outbound.CustomerRepository,
    createPaymentUC inbound.CreatePaymentUseCase,
    authorizePaymentUC inbound.AuthorizePaymentUseCase,
    capturePaymentUC inbound.CapturePaymentUseCase,
    cancelPaymentUC inbound.CancelPaymentUseCase,
    refundPaymentUC inbound.RefundPaymentUseCase,
) *PaymentService {
    return &PaymentService{
        companyRepo:        companyRepo,
        customerRepo:       customerRepo,
        createPaymentUC:    createPaymentUC,
        authorizePaymentUC: authorizePaymentUC,
        capturePaymentUC:   capturePaymentUC,
        cancelPaymentUC:    cancelPaymentUC,
        refundPaymentUC:    refundPaymentUC,
    }
}
```

Note: recebe use cases via interface inbound (`inbound.CreatePaymentUseCase`), nao implementacao concreta.

---

## Handler: Proto → Command → Result → Proto

```go
// internal/adapters/grpc/handler/payment_handler.go
func (s *PaymentService) CreatePayment(ctx context.Context, req *paymentProto.CreatePaymentRequest) (*paymentProto.PaymentResponse, error) {
    // 1. Proto → Command (mapping campo a campo)
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

    // 2. Chama use case
    res, err := s.createPaymentUC.Execute(ctx, cmd)
    if err != nil {
        return nil, status.Error(codes.Internal, err.Error())
    }

    // 3. Result → Proto via mapper
    return mapper.ToPaymentResponsePB(res), nil
}
```

---

## Handler Simples (1 campo)

```go
func (s *PaymentService) AuthorizePayment(ctx context.Context, req *paymentProto.AuthorizePaymentRequest) (*paymentProto.PaymentResponse, error) {
    res, err := s.authorizePaymentUC.Execute(ctx, payment.AuthorizePaymentCommand{
        PaymentID:      req.PaymentId,
        IdempotencyKey: req.IdempotencyKey,
    })
    if err != nil {
        return nil, status.Error(codes.Internal, err.Error())
    }
    return mapper.ToPaymentResponsePB(res), nil
}
```

Para commands simples, construa inline sem variavel separada.

---

## Error Handling no gRPC

Atualmente o projeto usa `codes.Internal` para todos os erros:

```go
if err != nil {
    return nil, status.Error(codes.Internal, err.Error())
}
```

Nao existe mapeamento DomainError → gRPC code ainda. Quando implementar, o padrao sera:

```go
func mapDomainErrorToGRPC(err error) error {
    var de *shared.DomainError
    if errors.As(err, &de) {
        switch de.Code {
        case shared.ErrCodeNotFound: return status.Error(codes.NotFound, de.Message)
        case shared.ErrCodeInvalidArgument: return status.Error(codes.InvalidArgument, de.Message)
        case shared.ErrCodeConflict: return status.Error(codes.AlreadyExists, de.Message)
        default: return status.Error(codes.Internal, de.Message)
        }
    }
    return status.Error(codes.Internal, err.Error())
}
```

---

## Regras

1. Handler faz APENAS: proto → command, execute, result → proto
2. ZERO logica de negocio no handler
3. ZERO validacao no handler — domain valida
4. handler e mapper sao packages separados
5. Conversao proto int32 → Go int: `int(req.Card.ExpirationMonth)`
6. Campos opcionais: `if req.Card != nil { cmd.Card = ... }`
7. Use case recebido via interface (inbound port)
