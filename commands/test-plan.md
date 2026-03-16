# Test Plan

Analisar mudancas do branch e gerar plano de testes com cobertura identificada e gaps.

## Usage

```
/test-plan [--write]
```

`--write`: alem de planejar, escreve os testes faltantes.

<arguments>
$ARGUMENTS
</arguments>

## Constraints

| Regra | Detalhes |
| --- | --- |
| Foco | Apenas codigo alterado no branch |
| Framework | Testify (require/assert), spies manuais |
| Package | Mesmo package (NAO `_test` separado) |
| Race | SEMPRE `-race` flag |
| Pattern | Table-driven para variantes, teste isolado para erros de infra |

## Progressive Context Loading

**ALWAYS** read `.claude/docs/back/14-TESTING-PATTERNS.md`
**IF** mudancas em domain **THEN** read `.claude/docs/back/03-AGGREGATES.md`
**IF** mudancas em application **THEN** read `.claude/docs/back/06-COMMANDS-RESULTS.md`

## Workflow

| Step | Action | Condition |
| --- | --- | --- |
| 1 | Coletar arquivos alterados | Sempre |
| 2 | Mapear codigo → testes existentes | Sempre |
| 3 | Identificar gaps de cobertura | Sempre |
| 4 | Gerar plano priorizado | Sempre |
| 5 | Escrever testes | Apenas se `--write` |
| 6 | Rodar testes | Apenas se `--write` |

### Step 1: Coletar

```bash
git diff origin/main...HEAD --name-only | grep '\.go$' | grep -v '_test\.go$'
```

### Step 2: Mapear

Para cada arquivo alterado, buscar:
- `{name}_test.go` no mesmo diretorio
- Testes existentes que cobrem funcoes alteradas

### Step 3: Identificar Gaps

Para cada funcao/metodo novo ou alterado:
- Tem teste?
- Happy path coberto?
- Error paths cobertos?
- Edge cases cobertos?
- Concorrencia testada (se aplicavel)?

### Step 4: Plano

```markdown
# Test Plan

## Resumo
- Arquivos alterados: {n}
- Com cobertura adequada: {n}
- Precisam testes: {n}
- Cenarios identificados: {n}

## Alta Prioridade

### {arquivo}
**Funcao**: `{FuncName}`
**Cenarios faltantes**:
- [ ] Happy path: {descricao}
- [ ] Erro: {condicao} → {DomainError esperado}
- [ ] Edge: {caso extremo}

**Estrutura sugerida**:
```go
func TestFuncName(t *testing.T) {
    tests := []struct {
        name     string
        // campos
        wantErr  bool
        wantCode string
    }{
        // casos
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // arrange, act, assert
        })
    }
}
```

## Media Prioridade
[...]

## Baixa Prioridade
[...]
```

### Step 5: Escrever (se --write)

Seguir estritamente:
- Spies manuais com Mutex (race-safe)
- `t.Helper()` em helpers
- `require.NoError()` para pre-condicoes
- `assert.Equal()` para verificacoes
- Table-driven para variantes
- Testes de erro de infra separados

### Step 6: Validar

```bash
go test ./... -race
```

## Validation

- [ ] Todo codigo alterado foi analisado
- [ ] Gaps identificados com prioridade
- [ ] Cenarios especificos (nao genericos)
- [ ] Se `--write`: testes passando com `-race`
