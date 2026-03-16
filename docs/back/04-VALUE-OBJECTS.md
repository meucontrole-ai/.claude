# Value Objects

## Quando Ativar
- Criar ou modificar Money, Method, Card, Pix, Boleto, Billing, Metadata
- Trabalhar com tipos do dominio que nao tem identidade

---

## Como Pensar

Value objects sao tipos sem identidade — dois Money(100, "BRL") sao iguais. Eles encapsulam validacao e comportamento. Nao use int64 direto para dinheiro; use `shared.Money`.

Principio: "Se posso comparar por valor e nao tem lifecycle, e value object."

---

## Money (Centavos, int64)

```go
// internal/domain/shared/money.go
type Currency string

const (
    CurrencyBRL Currency = "BRL"
    CurrencyUSD Currency = "USD"
    CurrencyEUR Currency = "EUR"
)

type Money struct {
    Amount   int64
    Currency Currency
}

func NewMoney(amount int64, currency Currency) (Money, error) {
    if amount < 0 {
        return Money{}, errors.New("money: amount must be non-negative")
    }
    if !validCurrencies[currency] {
        return Money{}, fmt.Errorf("money: unsupported currency %q", currency)
    }
    return Money{Amount: amount, Currency: currency}, nil
}

func MustNewMoney(amount int64, currency Currency) Money {
    m, err := NewMoney(amount, currency)
    if err != nil { panic(err) }
    return m
}

// Operacoes aritmeticas validam moeda
func (m Money) Add(other Money) (Money, error) { ... }
func (m Money) Subtract(other Money) (Money, error) { ... }
func (m Money) Multiply(factor int64) Money { ... }

// Comparacoes
func (m Money) IsZero() bool { return m.Amount == 0 }
func (m Money) IsPositive() bool { return m.Amount > 0 }
func (m Money) Equals(other Money) bool { ... }
func (m Money) GreaterThan(other Money) bool { ... }
func (m Money) LessThan(other Money) bool { ... }
```

Trade-off: `int64` centavos vs `decimal.Decimal`. Escolhemos int64 porque:
- Zero problemas de precisao de ponto flutuante
- Performance nativa (sem alocacao)
- Suficiente para valores em centavos (max ~92 quadrilhoes)
- Operacoes validam moeda em runtime

---

## Method (Union Discriminada)

```go
// internal/domain/payment/method.go
type Method struct {
    Type   MethodType
    Card   *CardData    // Ponteiro: nil se nao e card
    Pix    *PixData
    Boleto *BoletoData
}

func (m *Method) Validate() error {
    if !m.Type.IsValid() {
        return errors.New("payment method: invalid type")
    }
    populated := 0
    if m.Card != nil { populated++ }
    if m.Pix != nil { populated++ }
    if m.Boleto != nil { populated++ }
    if populated != 1 {
        return errors.New("payment method: exactly one method data must be populated")
    }
    // Valida o data especifico do tipo
    switch m.Type {
    case MethodTypeCard: return m.Card.Validate()
    case MethodTypePix: return m.Pix.Validate()
    case MethodTypeBoleto: return m.Boleto.Validate()
    }
    return nil
}
```

Trade-off: Union discriminada vs interface. Escolhemos struct com ponteiros porque:
- Numero fixo de metodos (card, pix, boleto) — nao precisa de extensibilidade
- Validacao centralizada em `Validate()`
- Serializacao simples (JSON/GORM)
- Go nao tem sum types nativos — ponteiro nil simula "variante ausente"

---

## Construtores de Method

```go
func NewCardMethod(card CardData) (Method, error) {
    if err := card.Validate(); err != nil {
        return Method{}, err
    }
    return Method{Type: MethodTypeCard, Card: &card}, nil
}

func NewPixMethod(pix PixData) (Method, error) {
    if err := pix.Validate(); err != nil {
        return Method{}, err
    }
    return Method{Type: MethodTypePix, Pix: &pix}, nil
}

func NewBoletoMethod(boleto BoletoData) (Method, error) {
    if err := boleto.Validate(); err != nil {
        return Method{}, err
    }
    return Method{Type: MethodTypeBoleto, Boleto: &boleto}, nil
}
```

Padrao: construtor valida → retorna (value, error). Nunca crie Method manualmente sem construtor.

---

## Metadata

```go
// internal/domain/shared/metadata.go
type Metadata map[string]string

func NewMetadata() Metadata { return make(Metadata) }

func (m Metadata) Set(key, value string) { m[key] = value }
func (m Metadata) Get(key string) (string, bool) { v, ok := m[key]; return v, ok }
func (m Metadata) Has(key string) bool { _, ok := m[key]; return ok }
func (m Metadata) Clone() Metadata { ... }
func (m Metadata) Merge(other Metadata) { ... }
```

Metadata e um map com metodos utilitarios. Inicializado com `NewMetadata()` para evitar nil map panic.

---

## Identity

```go
// internal/domain/shared/identity.go
func GenerateID() string {
    return uuid.New().String()
}
```

UUIDs v4 como IDs. Uma unica funcao, sem interface, sem abstração — simplicidade.

---

## Regras

1. Money SEMPRE em centavos (int64), NUNCA float
2. Operacoes de Money validam moeda — nao pode somar BRL + USD
3. Method exige exatamente UM campo preenchido (Card XOR Pix XOR Boleto)
4. Use construtores (`NewCardMethod`, `NewMoney`) — nao inicialize structs manualmente
5. Metadata inicializada com `NewMetadata()` para evitar nil map
6. Value objects sao imutaveis conceitualmente — Metadata e excecao pratica
