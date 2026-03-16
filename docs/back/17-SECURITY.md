# Security

## Quando Ativar
- Implementar webhook signature validation
- Gerenciar card tokens e dados sensiveis
- Implementar multi-tenancy scoping
- Configurar secrets e credenciais
- Auditar contra OWASP Top 10

---

## Como Pensar

Seguranca em sistema de pagamentos nao e feature opcional — e invariante. Cada decisao deve considerar: "se um atacante controlar esse input, o que acontece?"

Tres pilares:
1. **Validacao na fronteira**: webhook assinatura, input no domain construtor
2. **Isolamento por tenant**: MerchantID em toda query, idempotency scoped
3. **Minimo privilegio de dados**: armazene tokens, nunca PAN/CVV

Nao usamos middleware de validacao generico. A validacao vive nos construtores de dominio — fail-fast, explicita, testavel.

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

SEMPRE valide ANTES de processar payload. Use `hmac.Equal` (constant-time), NUNCA `==`.

---

## Card Token Handling

1. NUNCA armazene PAN completo ou CVV
2. SEMPRE use tokens do provider
3. Armazene apenas: token, brand, last4, expiry, funding
4. Validacao de expiracao no construtor `NewCardMethod()`

---

## Multi-tenancy

| Nivel | Campo | Isolamento |
|-------|-------|-----------|
| Merchant | `MerchantID` | Empresa que processa pagamentos |
| Workspace | `WorkspaceID` | Subdivisao do merchant |

Regras:
- Todo aggregate contem `MerchantID` + `WorkspaceID`
- Toda query filtra por `MerchantID`
- Customer scoped por `CompanyID`
- Idempotency key scoped por `MerchantID`

---

## Input Validation

A validacao principal acontece nos **construtores de dominio** (nao em middleware):

```go
func NewPayment(params CreateParams) (*Payment, error) {
    if params.MerchantID == "" {
        return nil, shared.NewDomainError(shared.ErrCodeInvalidArgument, "merchant_id is required")
    }
}
```

NAO usamos `go-playground/validator` com struct tags. Validacao explicita no construtor.

---

## Secrets Management

- Secrets via environment variables (`os.Getenv()`)
- NUNCA hardcode em codigo ou config files
- `.env` apenas em desenvolvimento local
- Credenciais por merchant ficam na entidade `Company`
- Em K8s: `Secret` resources com `secretRef`

---

## OWASP - Aplicacao ao Projeto

| Vulnerabilidade | Mitigacao |
|---|---|
| A01 Broken Access Control | Multi-tenancy scoping por MerchantID |
| A02 Cryptographic Failures | Card tokens, webhook HMAC |
| A03 Injection | GORM parameterized queries |
| A04 Insecure Design | Hexagonal arch, state machine |
| A05 Security Misconfiguration | Env-based config |
| A06 Vulnerable Components | govulncheck |
| A07 Auth Failures | Webhook signatures, idempotency |
| A08 Data Integrity | Optimistic locking, state machine |
| A09 Logging Failures | Structured logging sem PII |
| A10 SSRF | Provider URLs em env vars |

---

## Regras

1. NUNCA armazene PAN, CVV, ou dados de cartao em texto
2. SEMPRE valide assinatura de webhook antes de processar
3. Use `hmac.Equal` (constant-time), NUNCA `==` para comparar assinaturas
4. Todo aggregate contem MerchantID — isolamento obrigatorio
5. Validacao no domain construtor, NAO em middleware
6. Secrets via env vars, NUNCA hardcoded
7. Logs sem PII (sem CPF, cartao, tokens)
