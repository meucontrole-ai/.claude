# Documentacao de Aggregate

## Quando Ativar
- Documentar um aggregate root completo
- Gerar documentacao de entidade de dominio
- Pedidos como "documente esse agregado", "gere doc dessa entidade"

---

## Como Pensar

Documentacao de agregado responde: "qual a estrutura desse agregado, quais operacoes suporta, quais estados possui, e quais invariantes garante?"

Template RIGIDO e IMUTAVEL. Todo agregado segue a mesma estrutura.

---

## Template Obrigatorio

```markdown
# {NomeDoAggregate}

> {Uma frase descrevendo a entidade de negocio}

## Localizacao

`{caminho/completo/do/pacote/}`

## Arquivos

| Arquivo | Conteudo |
|---------|----------|
| `{arquivo.go}` | {o que contem} |

## Campos

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `{campo}` | `{tipo}` | {o que representa} |

## Construtor

| Funcao | Params | Retorno |
|--------|--------|---------|
| `{NewAggregate}` | `{ParamsStruct}` | `{(*Aggregate, error)}` |

### Validacoes do Construtor

| Campo Validado | Regra | Erro |
|----------------|-------|------|
| `{campo}` | {regra de validacao} | `{codigo + mensagem}` |

## Maquina de Estados

### Estados

| Status | Descricao |
|--------|-----------|
| `{StatusX}` | {significado de negocio} |

### Transicoes Validas

```
{StatusA} --> {StatusB}
{StatusA} --> {StatusC}
{StatusB} --> {StatusD}
...
```

## Mutacoes (Verbos de Negocio)

| Metodo | Pre-condicao | Efeitos | Evento Emitido |
|--------|-------------|---------|----------------|
| `{Authorize()}` | {status required} | {campos alterados} | `{EventName}` |

## Value Objects

| Tipo | Arquivo | Descricao |
|------|---------|-----------|
| `{Money}` | `{money.go}` | {o que representa} |

## Eventos de Dominio

| Evento | Constante | Emitido Quando |
|--------|-----------|----------------|
| `{PaymentCreatedEvent}` | `{EventPaymentCreated}` | {condicao} |

## Invariantes

- {Regra 1 que NUNCA pode ser violada}
- {Regra 2}
- {Regra 3}
```

---

## Regras de Preenchimento

1. LEIA todos os arquivos .go do pacote do agregado antes de documentar
2. Campos lista TODOS os campos da struct principal, sem excecao
3. Maquina de Estados vem do arquivo status.go ou equivalente
4. Transicoes Validas vem do mapa `validTransitions` real
5. Mutacoes lista TODOS os metodos publicos que alteram estado
6. Value Objects lista todos os tipos de valor usados pelo agregado
7. Eventos lista todos os eventos de dominio definidos no pacote
8. Invariantes sao as regras que o construtor e mutacoes garantem

---

## Secoes Opcionais (omitir se nao aplicavel)

- Maquina de Estados: omitir se o agregado nao tem status/transicoes
- Value Objects: omitir se nao ha value objects no pacote

---

## Regras

1. Template e IMUTAVEL — mesma estrutura para todo agregado
2. Toda informacao vem do codigo REAL
3. Leia TODOS os arquivos do pacote do agregado
4. Transicoes de estado devem corresponder EXATAMENTE ao mapa no codigo
5. Secoes opcionais sao omitidas inteiras, nunca vazias
6. Cada mutacao deve ter pre-condicao, efeitos e evento documentados
