# Health Check

Verificar se o projeto segue todas as convencoes documentadas em `.claude/docs/back/`.

## Constraints

| Regra | Detalhes |
| --- | --- |
| Escopo | Projeto inteiro |
| Output | Relatorio com compliance por categoria |
| Correcao | Apenas reportar, NAO corrigir (a menos que o usuario peca) |

## Progressive Context Loading

Carregar docs conforme analisa cada camada:
**IF** analisando domain **THEN** read `.claude/docs/back/03-AGGREGATES.md` + `04` + `05` + `07` + `13`
**IF** analisando application **THEN** read `.claude/docs/back/06-COMMANDS-RESULTS.md`
**IF** analisando adapters **THEN** read `.claude/docs/back/08` + `09` + `10` + `11`
**IF** analisando testes **THEN** read `.claude/docs/back/14-TESTING-PATTERNS.md`
**IF** analisando observability **THEN** read `.claude/docs/back/15` + `16`
**IF** analisando seguranca **THEN** read `.claude/docs/back/17-SECURITY.md`

## Workflow

| Step | Action |
| --- | --- |
| 1 | Verificar estrutura de diretorios |
| 2 | Verificar domain layer |
| 3 | Verificar application layer |
| 4 | Verificar adapters layer |
| 5 | Verificar testes |
| 6 | Verificar seguranca |
| 7 | Verificar observability |
| 8 | Gerar relatorio |

### Checks por Categoria

**Estrutura**
- [ ] Diretorios seguem hexagonal (domain/, ports/, application/, adapters/, bootstrap/, cmd/)
- [ ] Dependencia correta: adapters -> ports -> domain
- [ ] Nenhum import de adapter em domain
- [ ] Ports contem APENAS interfaces

**Domain**
- [ ] Construtores com fail-fast validation
- [ ] Mutacoes sao verbos de negocio (nao SetX)
- [ ] Status via state machine (transitionTo privado)
- [ ] DomainError para todos erros
- [ ] Value objects com construtores validados
- [ ] Events com campos especificos (nao aggregate inteiro)

**Application**
- [ ] Commands com apenas primitivos (sem struct tags)
- [ ] Results com apenas primitivos
- [ ] Use cases recebem interfaces (ports)
- [ ] Sequencia: validar → construir → persistir → publicar eventos

**Adapters**
- [ ] Handlers sem logica de negocio
- [ ] Conversao proto → command campo a campo
- [ ] Repository com optimistic locking (WHERE version = ?)
- [ ] Erros de infra wrapeados em DomainError
- [ ] Webhook com HMAC validation (hmac.Equal)

**Testes**
- [ ] Mesmo package (nao _test)
- [ ] Spies manuais (nao mocks gerados)
- [ ] Mutex em spies
- [ ] Table-driven para variantes
- [ ] Erros de infra em testes separados

**Seguranca**
- [ ] Nenhum PII em logs
- [ ] Sem PAN/CVV armazenado
- [ ] Secrets via env vars
- [ ] Multi-tenancy scoped (MerchantID em queries)

**Observability**
- [ ] Logger via context (logger.Ctx)
- [ ] Structured logging (sem Sprintf)
- [ ] Spans por camada com tracer names corretos
- [ ] Campos snake_case

### Relatorio

```markdown
# Health Check Report

## Score: {n}/100

| Categoria | Status | Score | Issues |
| --- | --- | --- | --- |
| Estrutura | OK/WARN/FAIL | {n}/15 | {count} |
| Domain | OK/WARN/FAIL | {n}/20 | {count} |
| Application | OK/WARN/FAIL | {n}/15 | {count} |
| Adapters | OK/WARN/FAIL | {n}/15 | {count} |
| Testes | OK/WARN/FAIL | {n}/15 | {count} |
| Seguranca | OK/WARN/FAIL | {n}/10 | {count} |
| Observability | OK/WARN/FAIL | {n}/10 | {count} |

## Issues Encontradas

### FAIL
- {issue com localizacao e correcao sugerida}

### WARN
- {issue menor}

## Recomendacoes
1. {prioridade 1}
2. {prioridade 2}
```
