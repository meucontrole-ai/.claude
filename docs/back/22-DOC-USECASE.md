# Documentacao de Use Case

## Quando Ativar
- Documentar um use case (application layer)
- Gerar documentacao de fluxo de negocio
- Pedidos como "documente esse caso de uso", "gere doc desse fluxo"

---

## Como Pensar

Documentacao de use case responde: "o que essa operacao faz passo a passo, quais dependencias injeta, quais erros pode retornar, e como se conecta ao resto do sistema?"

Template RIGIDO e IMUTAVEL. Todo use case segue a mesma estrutura.

---

## Template Obrigatorio

```markdown
# {NomeDoUseCase}

> {Uma frase descrevendo a operacao de negocio}

## Localizacao

`{caminho/completo/usecase.go}`

## Command de Entrada

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `{campo}` | `{tipo}` | {Sim/Nao} | {descricao} |

## Result de Saida

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `{campo}` | `{tipo}` | {descricao} |

## Dependencias Injetadas

| Interface | Porta | Responsabilidade |
|-----------|-------|-----------------|
| `{NomeInterface}` | `{inbound/outbound}` | {o que faz} |

## Fluxo de Execucao

1. {Passo 1 — validacao, busca, etc}
2. {Passo 2}
3. {Passo 3}
...

## Erros Possiveis

| Codigo | Condicao | Mensagem |
|--------|----------|----------|
| `{ErrCode}` | {quando ocorre} | `{mensagem}` |

## Eventos Emitidos

| Evento | Quando |
|--------|--------|
| `{EventName}` | {condicao de emissao} |

## Transicoes de Estado

{Se o use case altera status de um agregado}

```
{Estado A} --> {Estado B} [condicao]
```

## Observacoes

- {Nota 1 sobre comportamento especifico, idempotencia, etc}
- {Nota 2}
```

---

## Regras de Preenchimento

1. LEIA o arquivo usecase.go completo antes de documentar
2. LEIA o command e result correspondentes no domain
3. Command: liste TODOS os campos, sem excecao
4. Result: liste TODOS os campos, sem excecao
5. Fluxo de Execucao segue a ordem EXATA do codigo
6. Erros Possiveis lista TODOS os retornos de erro do use case
7. Eventos Emitidos vem do agregado — verifique quais mutacoes sao chamadas
8. Transicoes de Estado: omitir se o use case nao transiciona status

---

## Secoes Opcionais (omitir se nao aplicavel)

- Eventos Emitidos: omitir se nao ha publicacao de eventos
- Transicoes de Estado: omitir se nao ha mudanca de status
- Observacoes: omitir se nao ha comportamento especial

---

## Regras

1. Template e IMUTAVEL — mesma estrutura para todo use case
2. Toda informacao vem do codigo REAL
3. Leia usecase.go + command.go + result.go antes de gerar
4. Fluxo de Execucao deve ser preciso — cada passo corresponde a um bloco real de codigo
5. Nunca omita erros — todo `return err` ou `return shared.NewDomainError` deve estar na tabela
6. Secoes opcionais sao omitidas inteiras, nunca vazias
