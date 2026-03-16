# Code Review

Revisar codigo alterado no branch atual com foco em qualidade, bugs e padroes do projeto.

## Usage

```
/review [--scope=file|branch|staged]
```

Default: `branch` (todas as mudancas do branch vs main).

<arguments>
$ARGUMENTS
</arguments>

## Constraints

| Regra | Detalhes |
| --- | --- |
| Foco | Apenas codigo alterado, NAO a codebase inteira |
| Severidade | Critico (bloqueia) / Importante (deveria) / Sugestao (poderia) |
| Output | Markdown estruturado com findings numerados |
| Veredito | Aprovar / Aprovar com ressalvas / Rejeitar |

## Progressive Context Loading

**IF** mudancas em `domain/` **THEN** read `.claude/docs/back/03-AGGREGATES.md` + `.claude/docs/back/13-ERROR-HANDLING.md`
**IF** mudancas em `adapters/` **THEN** read `.claude/docs/back/08-GRPC-SERVICE.md` ou `.claude/docs/back/11-REPOSITORY-PATTERN.md` (conforme adapter)
**IF** mudancas em `application/` **THEN** read `.claude/docs/back/06-COMMANDS-RESULTS.md`
**IF** mudancas em testes **THEN** read `.claude/docs/back/14-TESTING-PATTERNS.md`

## Workflow

| Step | Action |
| --- | --- |
| 1 | Coletar diff do escopo selecionado |
| 2 | Ler docs relevantes (auto-routing por camada) |
| 3 | Ler contexto completo de cada arquivo alterado |
| 4 | Analisar contra 9 categorias |
| 5 | Gerar relatorio |

### Step 1: Coletar Diff

```bash
# branch (default)
git diff origin/main...HEAD

# staged
git diff --staged

# file
git diff -- <file>
```

### Step 2-3: Ler Contexto

Para cada arquivo alterado, ler o arquivo completo + arquivos relacionados:
- Use case → ler command + result + ports
- Adapter → ler interface implementada
- Teste → ler codigo testado
- Aggregate → ler value objects + events

### Step 4: Categorias de Analise

1. **Bugs** — erros logicos, nil dereference, race conditions
2. **Arquitetura** — violacao de camadas, dependencia invertida
3. **Code Smells** — duplicacao, complexidade, naming
4. **Error Handling** — DomainError correto, wrapping, propagacao
5. **Concorrencia** — mutex, goroutines sem ownership, -race
6. **Seguranca** — PII em logs, injection, HMAC comparison
7. **Performance** — queries N+1, alocacoes desnecessarias
8. **Testes** — cobertura das mudancas, spies corretos
9. **Nomenclatura** — Go idiomatico, consistencia

### Step 5: Relatorio

```markdown
# Code Review

## Resumo
[Semaforo] [Visao geral em 2 linhas]

## Scope
- Branch: {name}
- Arquivos: {count}
- Commits: {count}

## Findings

### Criticos
> ID-001 | {arquivo}:{linha} | {categoria}
> **Atual**: {codigo atual}
> **Sugestao**: {codigo sugerido}
> **Motivo**: {justificativa}

### Importantes
> ID-002 | ...

### Sugestoes
> ID-003 | ...

## Veredito
{Aprovar | Aprovar com ressalvas | Rejeitar} — {justificativa}
```

## Validation

- [ ] Todos os arquivos alterados foram revisados
- [ ] Findings tem ID, arquivo, linha, categoria
- [ ] Criticos tem sugestao de correcao
- [ ] Veredito coerente com findings
