# Documentacao de Package

## Quando Ativar
- Documentar um package Go completo
- Gerar documentacao de qualquer diretorio/pacote do projeto
- Pedidos como "documente esse pacote", "gere doc desse modulo"

---

## Como Pensar

Documentacao de package responde: "o que esse pacote faz, como ele se encaixa na arquitetura, e como usar?"

Cada documentacao segue um template RIGIDO e IMUTAVEL. Nao importa o pacote — a estrutura e sempre a mesma.

---

## Template Obrigatorio

Ao documentar um package, SEMPRE gere o seguinte formato, sem excecao:

```markdown
# {NomeDoPackage}

> {Uma frase descrevendo o proposito do pacote}

## Localizacao

`{caminho/completo/do/pacote}`

## Camada

{domain | application | ports | adapters | bootstrap | pkg | cmd}

## Responsabilidade

{2-4 frases descrevendo o que o pacote faz, que problema resolve e por que existe}

## Dependencias

| Depende de | Motivo |
|------------|--------|
| `{pacote}` | {por que depende} |

## Tipos Exportados

| Tipo | Categoria | Descricao |
|------|-----------|-----------|
| `{NomeTipo}` | {struct/interface/enum/type alias} | {O que representa} |

## Funcoes Exportadas

| Funcao | Assinatura | Descricao |
|--------|-----------|-----------|
| `{NomeFuncao}` | `{func(params) retorno}` | {O que faz} |

## Metodos Exportados

| Receiver | Metodo | Descricao |
|----------|--------|-----------|
| `{Tipo}` | `{Metodo(params) retorno}` | {O que faz} |

## Fluxo Interno

{Descricao numerada do fluxo principal de execucao do pacote, se aplicavel}

1. {Passo 1}
2. {Passo 2}
3. {Passo 3}

## Invariantes

- {Regra 1 que NUNCA pode ser violada}
- {Regra 2}

## Exemplo de Uso

{Bloco de codigo Go mostrando uso tipico}
```

---

## Regras de Preenchimento

1. LEIA todo o codigo do pacote antes de documentar — nunca invente
2. TODOS os tipos exportados devem aparecer na tabela — sem excecao
3. TODAS as funcoes exportadas devem aparecer — sem excecao
4. TODOS os metodos exportados devem aparecer — sem excecao
5. Dependencias listam APENAS imports internos do projeto, nao stdlib
6. Se o pacote nao tem Metodos Exportados, omita a secao inteira
7. Se o pacote nao tem Fluxo Interno claro, omita a secao inteira
8. Invariantes sao regras de negocio ou tecnicas que o pacote garante
9. Exemplo de Uso mostra o caso mais comum, com codigo REAL do projeto
10. NUNCA adicione secoes extras alem do template
11. NUNCA mude a ordem das secoes
12. NUNCA use emojis

---

## Secoes Opcionais (omitir se nao aplicavel)

- Metodos Exportados: omitir se o pacote so tem funcoes livres
- Fluxo Interno: omitir se o pacote e apenas tipos/constantes
- Exemplo de Uso: omitir se o pacote e interno e nao tem uso direto

---

## Regras

1. Template e IMUTAVEL — mesma estrutura para todo package, sem excecao
2. Toda informacao vem do codigo REAL — nunca suponha
3. Leia TODOS os arquivos .go do pacote antes de gerar
4. Secoes opcionais sao omitidas inteiras, nunca deixadas vazias
5. Descricoes sao tecnicas, concisas, sem adjetivos desnecessarios
6. Assinaturas de funcao sao simplificadas (omitir context.Context se obvio, manter tipos de retorno)
