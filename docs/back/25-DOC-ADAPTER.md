# Documentacao de Adapter

## Quando Ativar
- Documentar um adapter (repository, provider, consumer, publisher)
- Gerar documentacao de implementacao tecnica
- Pedidos como "documente esse adapter", "gere doc desse repositorio", "gere doc desse provider"

---

## Como Pensar

Documentacao de adapter responde: "qual interface implementa, como traduz entre mundo externo e domain, e quais decisoes tecnicas foram tomadas?"

Template RIGIDO e IMUTAVEL. Todo adapter segue a mesma estrutura.

---

## Template Obrigatorio

```markdown
# {NomeDoAdapter}

> {Uma frase descrevendo a implementacao tecnica}

## Localizacao

`{caminho/completo/do/pacote/}`

## Interface Implementada

| Interface | Porta | Arquivo da Interface |
|-----------|-------|---------------------|
| `{NomeInterface}` | `{ports/outbound/arquivo.go}` | `{metodos}` |

## Struct Principal

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `{campo}` | `{tipo}` | {o que faz} |

## Construtor

`{NewPostgresPaymentRepository(db *gorm.DB) *PostgresPaymentRepository}`

## Metodos Implementados

| Metodo | Descricao | Detalhes Tecnicos |
|--------|-----------|-------------------|
| `{Save(ctx, entity)}` | {o que faz} | {queries, serialization, etc} |

## Mapeamento Domain <-> Externo

| Direcao | Funcao | De | Para |
|---------|--------|----|------|
| Domain -> Externo | `{ToModel()}` | `{payment.Payment}` | `{PaymentModel}` |
| Externo -> Domain | `{ToDomain()}` | `{PaymentModel}` | `{payment.Payment}` |

## Modelo Externo (se aplicavel)

| Campo | Tipo DB/Proto | Tag | Descricao |
|-------|---------------|-----|-----------|
| `{campo}` | `{tipo}` | `{gorm:"..."}` | {mapeamento} |

## Decisoes Tecnicas

| Decisao | Motivo |
|---------|--------|
| `{decisao}` | {por que} |

## Tratamento de Erros

| Erro Externo | Mapeamento Domain | Codigo |
|--------------|-------------------|--------|
| `{gorm.ErrRecordNotFound}` | `{DomainError}` | `{NOT_FOUND}` |

## Observabilidade

| Tipo | Implementacao |
|------|--------------|
| Logging | {como loga} |
| Tracing | {spans criados} |
| Metrics | {metricas emitidas} |
```

---

## Regras de Preenchimento

1. LEIA todos os arquivos .go do pacote do adapter antes de documentar
2. Interface Implementada deve referenciar o arquivo EXATO em ports/
3. Metodos lista TODOS os metodos que implementam a interface
4. Mapeamento mostra TODAS as funcoes de conversao
5. Modelo Externo lista todos os campos do model (GORM, proto, etc)
6. Decisoes Tecnicas documenta escolhas como JSONB, indices, serialization
7. Tratamento de Erros mostra como erros de infra viram DomainError

---

## Secoes Opcionais (omitir se nao aplicavel)

- Modelo Externo: omitir se nao ha model (ex: publisher que nao persiste)
- Observabilidade: omitir se o adapter nao tem instrumentacao
- Decisoes Tecnicas: omitir se nao ha decisoes notaveis

---

## Regras

1. Template e IMUTAVEL — mesma estrutura para todo adapter
2. Toda informacao vem do codigo REAL
3. Leia TODOS os arquivos do pacote do adapter
4. Secoes opcionais sao omitidas inteiras, nunca vazias
5. Mapeamento de erros deve cobrir TODOS os erros tratados
6. Sempre identifique qual interface de ports/ o adapter implementa
