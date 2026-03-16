# Optimistic Locking & Idempotency

## Quando Ativar
- Implementar update concorrente seguro
- Trabalhar com idempotency keys
- Resolver conflitos de versao
- Prevenir processamento duplicado

---

## Como Pensar

Em sistema de pagamentos, dois webhooks podem chegar simultaneamente para o mesmo pagamento. Sem protecao, um sobrescreve o outro.

Optimistic locking: "assuma que ninguem mais esta editando, mas verifique na hora de salvar." Se a versao mudou, outra operacao foi mais rapida — rejeite e retente.

Idempotency: "se a mesma operacao chega duas vezes, retorne o mesmo resultado sem reexecutar."

---

## Optimistic Locking no Agregado

```go
// internal/domain/payment/payment.go
type Payment struct {
    // ...
    Version   int       // Incrementado a cada mutacao
    UpdatedAt time.Time // Atualizado a cada mutacao
}

func (p *Payment) touch() {
    p.UpdatedAt = time.Now().UTC()
    p.Version++
}
```

Toda mutacao (`Authorize`, `Capture`, etc.) chama `touch()`. Version comeca em 1.

---

## Optimistic Locking no Repositorio

```go
func (r *PostgresPaymentRepository) Update(ctx context.Context, p *payment.Payment) error {
    model := ToModel(p)
    currentVersion := model.Version
    model.Version = currentVersion + 1

    result := r.db.WithContext(ctx).
        Model(&PaymentModel{}).
        Where("id = ? AND version = ?", model.ID, currentVersion).
        Updates(model)

    if result.RowsAffected == 0 {
        return shared.NewDomainError(shared.ErrCodeConflict,
            fmt.Sprintf("optimistic lock conflict: payment %s version %d is stale", p.ID, currentVersion))
    }

    p.Version = model.Version  // Sync version back to domain
    return nil
}
```

`WHERE version = ?` + `RowsAffected == 0` = conflito detectado.

---

## Idempotency Key

```go
// No use case, ANTES de qualquer processamento:
existing, err := uc.paymentRepo.FindByIdempotencyKey(ctx, cmd.MerchantID, cmd.IdempotencyKey)
if err == nil && existing != nil {
    span.AddEvent("idempotency_hit")
    return appshared.ToPaymentResult(existing), nil  // Retorna o existente
}
```

Se ja existe pagamento com mesma (MerchantID, IdempotencyKey), retorna o resultado existente sem reprocessar.

No banco: `uniqueIndex:idx_payments_idempotency` garante unicidade.

---

## Regras

1. `touch()` incrementa Version + atualiza UpdatedAt em TODA mutacao
2. Update usa `WHERE id = ? AND version = ?` — nao confia em GORM auto-update
3. `RowsAffected == 0` = conflito de versao → retorna `ErrCodeConflict`
4. Idempotency key scoped por MerchantID (nao global)
5. Idempotency check e a PRIMEIRA coisa no use case
6. Duplicate key error no Save → `ErrCodeConflict` (nao panic)
