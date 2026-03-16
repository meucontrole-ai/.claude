# Webhook Service

## Quando Ativar
- Receber notificacoes de payment providers
- Implementar novo webhook handler
- Validar assinatura de webhook
- Processar status update de provider

---

## Como Pensar

Webhooks sao o caminho inverso: o provider nos notifica. O fluxo e:
1. Provider envia HTTP POST
2. Middleware valida assinatura (HMAC)
3. Parser extrai payload
4. Handler converte para domain command
5. Use case (`webhook_processor`) aplica `ApplyProviderStatus()`

Seguranca primeiro: NUNCA processe payload sem validar assinatura.

---

## Estrutura

```
internal/adapters/webhook/
  handlers.go                    # Router de webhooks
  matera_http.go                 # HTTP handler Matera
  matera_payment_handler.go      # Logica especifica Matera
  middleware.go                  # Middleware de assinatura
  payload_parser.go              # Parser de payloads
  event_types.go                 # Tipos de eventos webhook
```

Usa Fiber (HTTP framework), nao gRPC. Binario separado: `cmd/service_webhook/`.

---

## Webhook Signature Validation

```go
// ports/outbound/webhook_signature_validator.go
type WebhookSignatureValidator interface {
    Validate(payload []byte, signature string) error
}

// adapters/providers/matera/webhook_validator.go
func (v *MateraWebhookValidator) Validate(payload []byte, signature string) error {
    mac := hmac.New(sha256.New, v.secret)
    mac.Write(payload)
    expected := hex.EncodeToString(mac.Sum(nil))
    if !hmac.Equal([]byte(expected), []byte(signature)) {
        return shared.NewDomainError(shared.ErrCodeInvalidArgument, "invalid webhook signature")
    }
    return nil
}
```

HMAC-SHA256 com constant-time comparison (`hmac.Equal`). Nunca use `==` para comparar assinaturas.

---

## Fluxo Webhook → Domain

```
HTTP POST → Signature Validation → Parse Payload → ProviderStatus mapping
    → webhookprocessor.Execute() → payment.ApplyProviderStatus() → Save
```

---

## Regras

1. SEMPRE valide assinatura ANTES de processar payload
2. Use `hmac.Equal` (constant-time), NUNCA `==`
3. Webhook service e binario separado (service_webhook)
4. Status do provider e mapeado para `ProviderStatus` do domain
5. `ApplyProviderStatus` lida com idempotencia (webhook duplicado = no-op)
6. Logue payload recebido SEM dados sensiveis
