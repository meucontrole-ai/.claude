# Generate AI Context Documentation

Analisar qualquer repositorio e gerar uma estrutura `.claude/` completa com documentacao otimizada para IA.

## Usage

```
/doc-context [path-or-info]
```

<arguments>
$ARGUMENTS
</arguments>

## Constraints

| Regra | Detalhes |
| --- | --- |
| Output | Arquivos dentro de `.claude/` |
| Formato | Markdown otimizado para tokens (tabelas > prosa) |
| Tamanho | CLAUDE.md < 200 linhas, cada doc < 150 linhas |
| Idioma | Mesmo idioma do projeto |
| Precisao | Toda info baseada no codigo REAL, nunca inventar |

## Workflow

| Step | Action | Condition |
| --- | --- | --- |
| 1 | Descoberta do repositorio | Sempre |
| 2 | Identificar padroes e convencoes | Sempre |
| 3 | Perguntar ao usuario (5-8 perguntas) | Sempre |
| 4 | Gerar CLAUDE.md | Sempre |
| 5 | Gerar docs de referencia | Sempre |
| 6 | Gerar commands uteis | Se aplicavel |
| 7 | Validar estrutura | Sempre |

### Step 1: Descoberta

Analisar:
- Estrutura de diretorios (arquitetura)
- Arquivos de dependencia (package.json, go.mod, requirements.txt, Cargo.toml)
- Frameworks, linguagem, stack
- Padroes de teste existentes
- CI/CD configs
- README e docs existentes

### Step 2: Identificar Padroes

Extrair:
- Convencoes de nomenclatura
- Padroes de design (MVC, DDD, Clean Arch, etc.)
- Fluxo de dados
- Tratamento de erros
- Logging/observability
- Regras de negocio criticas

### Step 3: Perguntar ao Usuario

Fazer 5-8 perguntas sobre:
- Proposito do projeto e contexto de negocio
- Convencoes nao obvias (gotchas)
- Decisoes arquiteturais e motivacoes
- Regras criticas que nao sao obvias do codigo
- Integracoes externas

### Step 4: Gerar CLAUDE.md

Criar `.claude/CLAUDE.md` com:
- Auto-Routing obrigatorio
- Tabela IF/THEN de progressive loading
- Overview do repositorio (stack, estrutura, servicos)
- Commands (build, test, lint, deploy)
- Important Notes (gotchas criticos)
- Best Practices
- Self-Correction Protocol

### Step 5: Gerar Docs

Criar `.claude/docs/` com um arquivo por tema identificado. Cada doc segue:

```markdown
# {Topic Name}

{One-liner: o que este doc cobre}

## Quick Reference

| Conceito | Padrao |
| --- | --- |
| {item} | {como fazer} |

## Detalhes

[Conteudo especifico, com exemplos de codigo do projeto real]

## Anti-Patterns

- NAO faca X — faca Y
- NAO faca Z — faca W

## Checklist

- [ ] Verificacao 1
- [ ] Verificacao 2
```

### Step 6: Gerar Commands

Criar `.claude/commands/` com commands uteis detectados:
- Se tem testes → `/test-plan`
- Se tem PRs → `/create-pr`
- Se tem domain → `/scaffold`
- Se tem deploy → `/deploy-check`

### Step 7: Validar

- [ ] CLAUDE.md < 200 linhas
- [ ] Cada doc referenciado existe
- [ ] Tabela IF/THEN cobre todos os docs
- [ ] Token costs estimados
- [ ] Nenhuma info inventada — tudo do codigo real
