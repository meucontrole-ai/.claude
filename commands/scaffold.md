# Scaffold Domain Entity

Gerar estrutura completa de uma nova entidade seguindo os padroes do projeto.

## Usage

```
/scaffold <type> <name>
```

Tipos: `aggregate`, `usecase`, `adapter-grpc`, `adapter-repo`, `adapter-provider`, `value-object`, `event`

<arguments>
$ARGUMENTS
</arguments>

## Constraints

| Regra | Detalhes |
| --- | --- |
| Camada | Respeitar dependencia: adapters -> ports -> domain |
| Construtor | Fail-fast com validacao completa |
| Erros | Apenas DomainError (6 codigos) |
| Naming | Go idiomatico, PascalCase exports, camelCase internals |
| Testes | Criar junto, mesmo package, spies manuais |

## Progressive Context Loading

**IF** type = `aggregate` **THEN** read `.claude/docs/back/03-AGGREGATES.md` + `.claude/docs/back/04-VALUE-OBJECTS.md` + `.claude/docs/back/07-STATE-MACHINE.md`
**IF** type = `usecase` **THEN** read `.claude/docs/back/06-COMMANDS-RESULTS.md` + `.claude/docs/back/00-WALKTHROUGH.md`
**IF** type = `adapter-grpc` **THEN** read `.claude/docs/back/08-GRPC-SERVICE.md`
**IF** type = `adapter-repo` **THEN** read `.claude/docs/back/11-REPOSITORY-PATTERN.md` + `.claude/docs/back/12-OPTIMISTIC-LOCKING.md`
**IF** type = `adapter-provider` **THEN** read `.claude/docs/back/20-SDK-DEPENDENCIES.md`
**IF** type = `value-object` **THEN** read `.claude/docs/back/04-VALUE-OBJECTS.md`
**IF** type = `event` **THEN** read `.claude/docs/back/05-DOMAIN-EVENTS.md`

## Workflow

| Step | Action |
| --- | --- |
| 1 | Ler docs relevantes (auto-routing) |
| 2 | Analisar estrutura existente no projeto para manter consistencia |
| 3 | Criar arquivos do domain (tipos, construtor, validacoes) |
| 4 | Criar port/interface se necessario |
| 5 | Criar adapter/implementacao se aplicavel |
| 6 | Criar testes |
| 7 | Registrar no bootstrap/composition root se necessario |
| 8 | Verificar build e testes |

### Aggregate Scaffold

Arquivos gerados:
```
domain/{name}/
├── {name}.go           -- aggregate root (construtor, mutacoes, validacoes)
├── {name}_test.go      -- testes do aggregate
├── events.go           -- domain events
├── commands.go          -- commands (DTOs entrada)
├── results.go           -- results (DTOs saida)
└── errors.go            -- erros especificos (usando DomainError)

ports/inbound/
└── {name}_usecase.go    -- interface do use case

ports/outbound/
└── {name}_repository.go -- interface do repository
```

### Use Case Scaffold

```
application/{name}/
├── usecase.go           -- orquestrador (tracing, idempotency, persist, events)
└── usecase_test.go      -- testes com spies
```

### Adapter gRPC Scaffold

```
adapters/grpc/{name}/
├── handler/
│   └── handler.go       -- proto -> command -> use case -> result -> proto
└── mapper/
    └── mapper.go        -- conversoes domain <-> proto
```

### Adapter Repository Scaffold

```
adapters/repository/postgres/{name}/
├── repository.go        -- implementacao com GORM
├── model.go             -- GORM model + tags
└── mapper.go            -- ToModel() + ToDomain()
```

## Validation

```bash
go build ./...
go test ./... -race
```

- [ ] Build sem erros
- [ ] Testes passando
- [ ] Dependencias corretas (domain nao importa adapters)
- [ ] Registrado no bootstrap
