# Documentacao de Endpoint

## Quando Ativar
- Documentar um endpoint gRPC ou HTTP
- Gerar documentacao de API
- Pedidos como "documente esse endpoint", "gere doc dessa rota"

---

## Como Pensar

Documentacao de endpoint responde: "como chamar essa operacao externamente, qual o contrato de entrada/saida, e o que acontece por baixo?"

Template RIGIDO e IMUTAVEL. Todo endpoint segue a mesma estrutura.

---

## Template Obrigatorio

```markdown
# {NomeDoEndpoint}

> {Uma frase descrevendo a operacao exposta}

## Protocolo

{gRPC | HTTP}

## Localizacao do Handler

`{caminho/completo/handler.go}`

## Rota / Metodo RPC

- **gRPC**: `{service}/{Method}` (ex: `PaymentService/CreatePayment`)
- **HTTP**: `{METODO} {/rota}` (ex: `POST /webhooks/matera`)

## Request

| Campo | Tipo Proto/JSON | Obrigatorio | Descricao |
|-------|-----------------|-------------|-----------|
| `{campo}` | `{tipo}` | {Sim/Nao} | {descricao} |

## Response

| Campo | Tipo Proto/JSON | Descricao |
|-------|-----------------|-----------|
| `{campo}` | `{tipo}` | {descricao} |

## Mapeamento

```
Request Proto/JSON
    --> Command (domain)
        --> Use Case (application)
            --> Result (domain)
                --> Response Proto/JSON
```

| Etapa | Funcao/Metodo | Arquivo |
|-------|---------------|---------|
| Request -> Command | `{funcao}` | `{arquivo}` |
| Result -> Response | `{funcao}` | `{arquivo}` |

## Erros / Status Codes

| Codigo gRPC / HTTP Status | Condicao | Mensagem |
|---------------------------|----------|----------|
| `{codigo}` | {quando} | `{msg}` |

## Middlewares / Interceptors

| Middleware | Responsabilidade |
|------------|-----------------|
| `{nome}` | {o que faz} |

## Exemplo de Chamada

{Bloco de codigo com exemplo real de request/response}
```

---

## Regras de Preenchimento

1. LEIA o handler completo antes de documentar
2. LEIA o mapper correspondente (proto -> domain e domain -> proto)
3. Request e Response listam TODOS os campos do proto/JSON
4. Mapeamento mostra a cadeia completa de transformacao
5. Erros listam TODOS os codigos de erro possiveis
6. Middlewares listam todos que se aplicam a esse endpoint
7. Exemplo usa formato grpcurl (gRPC) ou curl (HTTP)

---

## Secoes Opcionais (omitir se nao aplicavel)

- Middlewares / Interceptors: omitir se nao ha middleware especifico

---

## Regras

1. Template e IMUTAVEL — mesma estrutura para todo endpoint
2. Toda informacao vem do codigo REAL e do proto/schema
3. Leia handler + mapper + proto antes de gerar
4. Campos de Request/Response devem corresponder 1:1 ao proto ou schema JSON
5. Secoes opcionais sao omitidas inteiras, nunca vazias
6. Exemplo de chamada deve ser funcional — copiavel e executavel
