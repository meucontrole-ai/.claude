# Commands & Results

## Quando Ativar
- Criar novo command de entrada para use case
- Criar novo result de saida
- Entender fronteira entre adapter e domain

---

## Como Pensar

Commands sao DTOs de entrada. Results sao DTOs de saida. Ambos vivem em `domain/` porque representam o contrato do use case — independente de protocolo (gRPC, HTTP, CLI).

Regra fundamental: commands usam APENAS tipos primitivos (`string`, `int64`, `int`, `bool`). Nenhum tipo de dominio rico. Isso porque o adapter (gRPC handler) precisa construir o command sem conhecer o domain internamente.

A conversao primitivo → tipo rico acontece no use case:
- Handler: proto → `CreatePaymentCommand` (primitivos)
- Use case: `cmd.Amount` (int64) → `shared.NewMoney(cmd.Amount, cmd.Currency)` (Money)

---

## Command Real

```go
// internal/domain/payment/command.go
type CreatePaymentCommand struct {
    MerchantID         string
    WorkspaceID        string
    IdempotencyKey     string
    Amount             int64       // Centavos, nao Money
    Currency           string      // "BRL", nao shared.Currency
    CaptureMethod      string      // "automatic", nao CaptureMethod enum
    ConfirmationMethod string
    MethodType         string      // "card", nao MethodType enum
    Card               *CardInput
    Pix                *PixInput
    Boleto             *BoletoInput
    Billing            BillingInput
    Splits             []SplitInput
    FraudData          *FraudDataInput
    Description        string
    Metadata           shared.Metadata
}

type CardInput struct {
    Token           string
    Brand           string
    LastFourDigits  string
    ExpirationMonth int
    ExpirationYear  int
    HolderName      string
    Funding         string
    Installments    int
    BinNumber       string
}
```

Note: ZERO tags de struct (nem json, nem validate, nem gorm). Commands sao structs puras.

---

## Outros Commands

```go
type AuthorizePaymentCommand struct {
    PaymentID      string
    IdempotencyKey string
}

type CapturePaymentCommand struct {
    PaymentID      string
    Amount         int64
    IdempotencyKey string
}

type CancelPaymentCommand struct {
    PaymentID      string
    Reason         string
    IdempotencyKey string
}
```

Padrao: `{Acao}PaymentCommand` com apenas os campos necessarios.

---

## Result Real

```go
// internal/domain/payment/result.go
type PaymentResult struct {
    ID                    string
    Status                string
    Amount                int64
    Currency              string
    CapturedAmount        int64
    RefundedAmount        int64
    ProviderName          string
    ProviderTransactionID string
    PixQRCode             string
    PixQRCodeURL          string
    PixQRCodeBase64       string
    BoletoBarcode         string
    BoletoURL             string
    Metadata              shared.Metadata
    CreatedAt             string    // Formatado como RFC3339
}
```

Results tambem usam primitivos. A conversao Domain → Result acontece em `application/shared/mapper.go`:

```go
// internal/application/shared/mapper.go
func ToPaymentResult(p *payment.Payment) *payment.PaymentResult {
    result := &payment.PaymentResult{
        ID:       p.ID,
        Status:   p.Status.String(),
        Amount:   p.Amount.Amount,
        Currency: string(p.Amount.Currency),
        // ...
        CreatedAt: p.CreatedAt.Format("2006-01-02T15:04:05Z"),
    }

    if p.Method.Pix != nil {
        result.PixQRCode = p.Method.Pix.QRCode
        result.PixQRCodeURL = p.Method.Pix.QRCodeURL
    }
    return result
}
```

---

## Fluxo Completo

```
Proto Request → gRPC Handler (construi Command) → Use Case (converte para tipos ricos)
                                                           ↓
Proto Response ← gRPC Mapper (Result → Proto)  ← Use Case (retorna Result)
```

---

## Trade-off: Commands em domain/ vs ports/

| Aspecto | Em domain/ (nosso) | Em ports/ |
|---|---|---|
| Quem pode usar | domain, application, adapters | ports, application, adapters |
| Acoplamento | Command e do contexto do agregado | Command e do contrato |
| Localizacao | Junto do agregado que consome | Separado |

Escolhemos domain/ porque o command define o vocabulario da operacao de negocio, nao um contrato tecnico.

---

## Regras

1. Commands usam APENAS primitivos — string, int64, int, bool, ponteiros para structs de input
2. Results usam APENAS primitivos — mesma regra
3. ZERO struct tags (nem json, nem validate)
4. Commands vivem em `domain/<aggregate>/command.go`
5. Results vivem em `domain/<aggregate>/result.go`
6. Conversao primitivo → tipo rico acontece NO USE CASE, nao no handler
7. Conversao agregado → result acontece em `application/shared/mapper.go`
8. Metadata (shared.Metadata) e a unica excecao — e um map[string]string, ja primitivo
