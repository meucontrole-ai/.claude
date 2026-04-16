# Repository Pattern

## Quando Ativar
- Criar novo repositorio
- Modificar persistencia (save, update, find)
- Trabalhar com GORM models e mappers
- Implementar optimistic locking no repositorio

---

## Como Pensar

O repositorio traduz entre domain e banco de dados. Ele conhece GORM, SQL e PostgreSQL; o domain nao sabe que existe um banco.

Tres camadas de conversao:
1. **Domain aggregate** (`payment.Payment`) — regras de negocio
2. **GORM model** (`PaymentModel`) — struct com tags GORM
3. **Tabela PostgreSQL** — schema real

Mappers (`ToModel`, `ToDomain`) convertem entre 1 e 2. GORM converte entre 2 e 3.

---

## Outbound Port (Interface)

```go
// internal/ports/outbound/payment_repository.go
type PaymentRepository interface {
    Save(ctx context.Context, p *payment.Payment) error
    Update(ctx context.Context, p *payment.Payment) error
    FindByID(ctx context.Context, id string) (*payment.Payment, error)
    FindByIdempotencyKey(ctx context.Context, merchantID, key string) (*payment.Payment, error)
    FindByProviderTransactionID(ctx context.Context, provider, txnID string) (*payment.Payment, error)
    ListByMerchant(ctx context.Context, merchantID string, limit, offset int) ([]*payment.Payment, error)
}
```

Interface limpa: recebe/retorna domain types. Nenhuma referencia a GORM.

---

## GORM Model

```go
// internal/adapters/repository/postgres/model.go
type PaymentModel struct {
    ID             string `gorm:"type:varchar(36);primaryKey"`
    MerchantID     string `gorm:"type:varchar(36);not null;index:idx_payments_merchant"`
    WorkspaceID    string `gorm:"type:varchar(36);not null;index:idx_payments_workspace"`
    IdempotencyKey string `gorm:"type:varchar(255);not null;uniqueIndex:idx_payments_idempotency"`

    AmountCentavos         int64  `gorm:"not null"`
    AmountCurrency         string `gorm:"type:varchar(3);not null"`
    CapturedAmountCentavos int64  `gorm:"not null;default:0"`
    RefundedAmountCentavos int64  `gorm:"not null;default:0"`

    Status    string `gorm:"type:varchar(30);not null;index:idx_payments_status"`
    // ...

    MethodType string     `gorm:"type:varchar(20);not null;index:idx_payments_method_type"`
    MethodData JSONColumn `gorm:"type:jsonb"`  // Dados do metodo como JSONB

    Version  int            `gorm:"not null;default:1"`
    Attempts []AttemptModel `gorm:"foreignKey:PaymentID;references:ID"`
}

func (PaymentModel) TableName() string { return "payments" }
```

Padrao de desnormalizacao:
- `Money` → `AmountCentavos` + `AmountCurrency` (duas colunas)
- `Method` → `MethodType` + `MethodData` (JSONB)
- `Billing` → colunas planas (`BillingName`, `BillingEmail`...) + `BillingAddress` (JSONB)

---

## JSONColumn (Custom Type)

```go
type JSONColumn map[string]string

func (j JSONColumn) Value() (driver.Value, error) {
    if j == nil { return nil, nil }
    return json.Marshal(j)
}

func (j *JSONColumn) Scan(value interface{}) error {
    if value == nil { *j = nil; return nil }
    bytes := value.([]byte)
    result := make(map[string]string)
    json.Unmarshal(bytes, &result)
    *j = result
    return nil
}
```

Implementa `driver.Valuer` e `sql.Scanner` para GORM serializar automaticamente.

---

## Mapper: Domain ↔ Model

```go
// internal/adapters/repository/postgres/mapper.go
func ToModel(p *payment.Payment) *PaymentModel {
    model := &PaymentModel{
        ID:                     p.ID,
        MerchantID:             p.MerchantID,
        AmountCentavos:         p.Amount.Amount,
        AmountCurrency:         string(p.Amount.Currency),
        Status:                 string(p.Status),
        MethodType:             string(p.Method.Type),
        MethodData:             marshalMethodData(p.Method),
        ProviderMetadata:       JSONColumn(p.ProviderMetadata),
        Version:                p.Version,
        // ... todos os campos
    }
    return model
}

func ToDomain(model *PaymentModel) *payment.Payment {
    currency := shared.Currency(model.AmountCurrency)
    p := &payment.Payment{
        ID:         model.ID,
        Amount:     shared.Money{Amount: model.AmountCentavos, Currency: currency},
        Status:     payment.Status(model.Status),
        Method:     unmarshalMethod(payment.MethodType(model.MethodType), model.MethodData),
        Version:    model.Version,
        // ...
    }
    return p
}
```

Note as conversoes de tipo: `string(p.Status)` para model, `payment.Status(model.Status)` para domain.

---

## Repository Implementation

```go
// internal/adapters/repository/postgres/payment_repository.go
func (r *PostgresPaymentRepository) Save(ctx context.Context, p *payment.Payment) error {
    ctx, span := startDBSpan(ctx, "INSERT", "payments")
    var err error
    defer func() { endDBSpan(span, err, 1) }()

    model := ToModel(p)
    result := r.db.WithContext(ctx).Create(model)
    if result.Error != nil {
        if isDuplicateKeyError(result.Error) {
            return shared.WrapDomainError(shared.ErrCodeConflict,
                fmt.Sprintf("payment already exists for idempotency key %q", p.IdempotencyKey),
                result.Error)
        }
        return shared.WrapDomainError(shared.ErrCodeInvariantViolation, "failed to save payment", result.Error)
    }
    return nil
}

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

    p.Version = model.Version  // Atualiza version no domain
    return nil
}

func (r *PostgresPaymentRepository) FindByID(ctx context.Context, id string) (*payment.Payment, error) {
    var model PaymentModel
    result := r.db.WithContext(ctx).
        Preload("Attempts", func(db *gorm.DB) *gorm.DB {
            return db.Order("attempt_number ASC")
        }).
        First(&model, "id = ?", id)

    if errors.Is(result.Error, gorm.ErrRecordNotFound) {
        return nil, shared.NewDomainError(shared.ErrCodeNotFound, fmt.Sprintf("payment %s not found", id))
    }
    return ToDomain(&model), nil
}
```

---

## Tracing no Repositorio

```go
func startDBSpan(ctx context.Context, op, table string) (context.Context, trace.Span) {
    // Cria span OpenTelemetry para cada operacao de DB
}
func endDBSpan(span trace.Span, err error, rowsAffected int64) {
    // Finaliza span com status e metricas
}
```

Todo metodo do repositorio cria um span de tracing.

---

## Regras

1. Domain type NUNCA aparece com tags GORM — use model separado
2. Mapper converte explicitamente (não reflection, não automapper)
3. `ToModel()` e `ToDomain()` são funções do package postgres, não métodos
4. Optimistic locking: `WHERE version = ?` + check `RowsAffected`
5. Duplicate key → `ErrCodeConflict`
6. Record not found → `ErrCodeNotFound`
7. Erros de infra → `WrapDomainError` (não retorne erro GORM puro)
8. Preload de Attempts com ORDER BY `attempt_number ASC`
9. `WithContext(ctx)` em TODA query GORM
10. Todo método do repo cria span de tracing

---

## Armadilha Crítica: `Updates(map)` Incompleto

Se você usa `Updates(map[string]interface{}{...})` pra escolher quais colunas mexer, **o map DEVE listar TODO campo mutável**. Esquecer um campo = mutação em memória NUNCA chega no banco, e o próximo `FindByID` devolve o valor antigo. Bug silencioso de perda de dados.

**Exemplo do bug:**
```go
// ❌ VERSÃO COM BUG — um campo mutável esquecido
result := r.db.WithContext(ctx).
    Model(&model.AggregateModel{}).
    Where("id = ? AND version = ?", m.ID, a.OriginalVersion).
    Updates(map[string]interface{}{
        "name":    m.Name,
        "status":  m.Status,
        "version": m.Version,
        // ❌ session_id esquecido! Domínio muta, in-memory reflete,
        // DB nunca persiste. Próximo FindByID devolve valor antigo.
    })
```

Sintoma típico: método de domínio muta um campo, a response HTTP do método até mostra o valor novo (porque veio do objeto em memória), mas o próximo read do DB devolve o valor antigo — operação idempotente se repete desnecessariamente, comportamentos "fantasma" aparecem, audit logs fragmentam.

**Checklist antes de commitar um repositório com `Updates(map)`:**

1. Liste TODOS os campos da struct de domínio que mudam via métodos de mutação.
2. Para cada um, confirme que está no map do `Updates`.
3. Se existe `touch()` / `version++` no domínio, garanta que `version` está no map.
4. Se o agregado tem `SessionID`, `Metadata`, `Snapshot`, ou qualquer campo de auditoria — incluí-los.
5. Considere `db.Save(&model)` quando quer update full-row — menos propenso a esse bug, mas não permite optimistic locking granular.

### Alternativa menos propensa a bug

```go
result := r.db.WithContext(ctx).
    Model(&model.TableModel{}).
    Where("id = ? AND version = ?", m.ID, t.OriginalVersion).
    Select("*").                     // força update de TODAS as colunas
    Omit("id", "created_at").        // menos as imutáveis
    Updates(&m)                       // passa o struct, não o map
```

Essa forma é auto-documentada (update everything except X) e não silenciosamente esquece campos novos adicionados ao modelo.
