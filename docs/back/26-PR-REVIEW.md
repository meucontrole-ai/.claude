# PR Review

## Quando Ativar
- Revisar codigo de uma branch antes de merge
- Pedidos como "revise o PR", "review da branch", "analise o diff", "revise as mudancas"
- Comparar branch atual com main/master

---

## Como Pensar

PR Review e auditoria tecnica completa. Voce age como revisor senior: executa comandos git para coletar TODOS os diffs, analisa cada arquivo alterado contra os padroes do projeto, busca bugs, smells e inconsistencias, e gera um relatorio estruturado com sugestoes de correcao que seguem o codigo existente.

NAO e review superficial. Voce DEVE ler cada linha alterada.

---

## Procedimento de Execucao

### Fase 1: Coleta de Dados (git)

Execute os seguintes comandos para entender o escopo completo da mudanca:

```bash
# 1. Identifica a branch base
git rev-parse --abbrev-ref HEAD

# 2. Encontra o merge-base com main/master
git merge-base HEAD main || git merge-base HEAD master

# 3. Lista todos os arquivos alterados desde a divergencia
git diff --name-status $(git merge-base HEAD main)..HEAD

# 4. Diff completo com contexto
git diff $(git merge-base HEAD main)..HEAD

# 5. Commits da branch (mensagens e escopo)
git log --oneline $(git merge-base HEAD main)..HEAD

# 6. Diff stat (resumo de linhas adicionadas/removidas por arquivo)
git diff --stat $(git merge-base HEAD main)..HEAD
```

### Fase 2: Leitura Completa

Para CADA arquivo alterado:
1. Leia o arquivo COMPLETO (nao apenas o diff) para entender o contexto
2. Se o arquivo e um use case, leia tambem o command, result e ports relacionados
3. Se o arquivo e um adapter, leia a interface que implementa
4. Se o arquivo e um teste, leia o codigo que esta testando

### Fase 3: Analise

Analise cada arquivo contra as seguintes categorias:

#### 3.1 Bugs e Erros Logicos
- Nil pointer dereference
- Race conditions (acesso concorrente sem mutex)
- Off-by-one errors
- Erros silenciados (err ignorado sem _ explicito)
- Recursos nao fechados (conexoes, files, channels)
- Deadlocks potenciais
- Panic em producao (index out of range, map nil)
- SQL injection, command injection
- Conversoes de tipo inseguras

#### 3.2 Violacoes de Arquitetura
- Domain importando adapters ou application
- Ports contendo implementacao (structs, DTOs)
- Commands/Results com tipos ricos (devem ser primitivos)
- Commands/Results com struct tags (devem ser limpos)
- Use case importando adapter diretamente
- Logica de negocio em adapter ou handler

#### 3.3 Code Smells
- Funcao > 500 linhas
- Arquivo > 500 linhas
- God struct (struct com muitas responsabilidades)
- Nested ifs > 3 niveis (falta early return)
- String magica repetida (deve ser constante)
- fmt.Sprintf em mensagens de log (deve ser structured)
- Logger avulso sem context (deve usar logger.Ctx(ctx))
- Mocks gerados ao inves de spies manuais
- Campos exportados sem Godoc

#### 3.4 Tratamento de Erros
- Erro de infra retornado puro (sem DomainError wrap)
- DomainError com codigo inexistente (fora dos 6 padroes)
- Erro sem contexto suficiente na mensagem
- Error log sem .Err(err)
- Erro ignorado sem justificativa

#### 3.5 Concorrencia
- Acesso a map compartilhado sem mutex
- Goroutine sem context cancellation
- Channel sem close
- sync.WaitGroup mal utilizado

#### 3.6 Seguranca
- Dados sensiveis em logs (PAN, CVV, CPF, tokens)
- Secrets hardcoded
- Input nao validado na fronteira do sistema
- CORS/auth bypass

#### 3.7 Performance
- Query N+1
- Alocacao desnecessaria em hot path
- Copia de struct grande em loop (deveria ser ponteiro)
- Regex compilada dentro de loop

#### 3.8 Testes
- Cenario critico sem cobertura
- Teste sem assert (teste que nunca falha)
- Teste com sleep (flaky)
- Spy sem mutex (race condition com -race)
- Falta de table-driven para variacoes

#### 3.9 Nomenclatura e Convencoes
- Package name diferente do padrao (deve ser lowercase junto)
- Diretorio diferente do padrao (deve ser snake_case)
- Funcao/struct fora do padrao PascalCase
- Interface com prefixo "I"
- Construtor sem prefixo "New"

---

## Template do Relatorio (Obrigatorio)

Gere um arquivo Markdown com o seguinte formato RIGIDO:

```markdown
# PR Review: {nome-da-branch}

> Revisao automatizada em {data}

## Resumo

- **Branch**: `{branch-atual}`
- **Base**: `{main/master}`
- **Commits**: {N} commits
- **Arquivos alterados**: {N} arquivos
- **Linhas**: +{adicionadas} / -{removidas}

## Escopo da Mudanca

{2-3 frases descrevendo o que o PR faz no nivel de negocio}

## Commits

| Hash | Mensagem |
|------|----------|
| `{hash}` | {mensagem} |

## Arquivos Alterados

| Arquivo | Status | Camada | +/- |
|---------|--------|--------|-----|
| `{caminho}` | {A/M/D} | {domain/application/adapters/...} | +{n}/-{n} |

---

## Findings

### Criticos (Bloqueia merge)

{Se nao houver findings criticos, escreva: "Nenhum finding critico encontrado."}

#### [{ID}] {Titulo curto do problema}

- **Arquivo**: `{caminho/arquivo.go}:{linha}`
- **Categoria**: {Bug | Violacao de Arquitetura | Seguranca}
- **Severidade**: Critico
- **Descricao**: {O que esta errado e por que e um problema}
- **Codigo atual**:
```go
{trecho do codigo problematico}
```
- **Sugestao de correcao**:
```go
{codigo corrigido seguindo os padroes do projeto}
```
- **Justificativa**: {Por que a correcao segue o padrao do projeto, referenciando skill ou codigo existente}

---

### Importantes (Deveria corrigir)

{Se nao houver, escreva: "Nenhum finding importante encontrado."}

#### [{ID}] {Titulo}

{Mesmo formato dos criticos, com Severidade: Importante}

---

### Sugestoes (Nice to have)

{Se nao houver, escreva: "Nenhuma sugestao adicional."}

#### [{ID}] {Titulo}

{Mesmo formato, com Severidade: Sugestao}

---

## Checklist de Conformidade

| Criterio | Status | Observacao |
|----------|--------|------------|
| Regra de dependencia (imports) | {OK/FALHA} | {detalhe} |
| DomainError para todos os erros | {OK/FALHA} | {detalhe} |
| Logging com logger.Ctx(ctx) | {OK/FALHA} | {detalhe} |
| Sem dados sensiveis em logs | {OK/FALHA} | {detalhe} |
| Structured logging (sem Sprintf) | {OK/FALHA} | {detalhe} |
| Commands com primitivos apenas | {OK/FALHA} | {detalhe} |
| Testes com spy (sem mocks gerados) | {OK/FALHA} | {detalhe} |
| Table-driven tests | {OK/FALHA} | {detalhe} |
| Race-safe (mutex em shared state) | {OK/FALHA} | {detalhe} |
| Godoc em exportados novos | {OK/FALHA} | {detalhe} |
| Nomenclatura conforme convencoes | {OK/FALHA} | {detalhe} |

## Veredito

{APROVADO | APROVADO COM RESSALVAS | MUDANCAS NECESSARIAS}

{1-2 frases finais com recomendacao}
```

---

## Regras de Identificacao de Findings

1. IDs sequenciais: `CR-01`, `CR-02` (criticos), `IM-01`, `IM-02` (importantes), `SG-01`, `SG-02` (sugestoes)
2. Cada finding referencia arquivo e linha EXATOS
3. Sugestao de correcao usa codigo REAL do projeto como referencia
4. Nunca invente padroes — busque no codigo existente como o problema foi resolvido antes
5. Se um padrao nao existe no projeto, referencie a skill correspondente

---

## Classificacao de Severidade

| Severidade | Criterio | Exemplos |
|------------|----------|----------|
| **Critico** | Bug confirmado, vulnerabilidade, ou violacao que causa falha em producao | Nil pointer, race condition, SQL injection, domain importando adapter |
| **Importante** | Violacao de padrao, smell que dificulta manutencao, ou erro potencial | Erro sem wrap, log sem context, teste sem assert, funcao > 500 linhas |
| **Sugestao** | Melhoria de legibilidade, performance nao-critica, convencao menor | Variavel com nome ruim, comentario desnecessario, import nao agrupado |

---

## Regras

1. SEMPRE execute os comandos git para coletar diffs — nunca assuma
2. LEIA cada arquivo alterado por COMPLETO, nao apenas o diff
3. Template e IMUTAVEL — mesma estrutura para toda review
4. Todo finding tem codigo atual + sugestao de correcao
5. Sugestoes de correcao seguem o modelo EXISTENTE do projeto
6. Se nao encontrou problemas numa categoria, registre como "OK" no checklist
7. Veredito e objetivo: se tem critico, e MUDANCAS NECESSARIAS
8. Relatorio e salvo como Markdown no diretorio raiz ou onde o usuario indicar
9. Nunca aprove com findings criticos pendentes
10. Findings duplicados (mesmo problema em varios lugares) devem ser agrupados em um unico finding com lista de ocorrencias
