# Infrastructure & Deploy

## Quando Ativar
- Criar/modificar Dockerfile, docker-compose
- Configurar LocalStack (AWS local)
- Criar/modificar K8s manifests
- Configurar CI/CD pipelines
- Configurar Makefile

---

## Como Pensar

Infra serve o codigo, nao o contrario. Decisoes:
- **Dev local**: docker-compose com PostgreSQL + LocalStack. Zero dependencia de cloud real
- **Producao**: containers leves (multi-stage), non-root, health checks
- **Multi-service**: cada servico e um binario independente com seu proprio entrypoint

Principio: "se eu deletar o cluster e subir de novo, tudo funciona automaticamente?"

---

## Multi-Service Architecture

| Servico | Binario | Protocolo | Funcao |
|---------|---------|-----------|--------|
| Core | `cmd/service_core/` | gRPC | API principal de pagamentos |
| Webhook | `cmd/service_webhook/` | HTTP (Fiber) | Receber notificacoes de providers |
| Saga | `cmd/service_saga/` | SQS Consumer | Validacao de pagamentos |
| CLI | `cmd/saga_cli/` | CLI | Testes manuais de saga |

---

## Dockerfile (Multi-stage)

```dockerfile
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git ca-certificates
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /app/server ./cmd/service_core

FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata && adduser -D -g '' appuser
COPY --from=builder /app/server /usr/local/bin/server
USER appuser
EXPOSE 50051
ENTRYPOINT ["server"]
```

---

## Docker Compose (Development)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: payment
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: payment_core
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U payment"]

  localstack:
    image: localstack/localstack
    environment:
      SERVICES: sqs,events
    # SQS + EventBridge para saga local

  grafana:
    image: grafana/grafana
    # Dashboards de metricas
```

---

## LocalStack (AWS Local)

- **SQS**: fila de mensagens para saga consumer
- **EventBridge**: publicacao de eventos de dominio

Configuracao via env vars: `AWS_ENDPOINT`, `AWS_REGION`, etc.

---

## Graceful Shutdown

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

grpcServer.GracefulStop()
// ou: srv.Shutdown(ctx) para HTTP
```

Timeout (15s) menor que K8s `terminationGracePeriodSeconds` (30s).

---

## Kubernetes

### Deployment
```yaml
spec:
  containers:
    - name: service-core
      resources:
        requests: { cpu: 100m, memory: 128Mi }
        limits: { cpu: 500m, memory: 512Mi }
      livenessProbe:
        httpGet: { path: /health/live, port: 8081 }
      readinessProbe:
        httpGet: { path: /health/ready, port: 8081 }
      envFrom:
        - configMapRef: { name: payment-core-config }
        - secretRef: { name: payment-core-secrets }
```

### HPA + PDB
- HPA: scale baseado em CPU (70% target)
- PDB: `minAvailable: 1` para zero-downtime deploys

---

## CI/CD (GitHub Actions)

```yaml
jobs:
  lint:     golangci-lint
  test:     go test ./internal/... -v -race
  security: gosec + govulncheck
  build:    docker build + push (apenas main)
```

---

## Regras

1. Container roda como non-root user
2. NUNCA tag `latest` em producao (use SHA ou semver)
3. Health checks separados: liveness (processo vivo) vs readiness (deps ok)
4. Secrets via K8s Secret + secretRef (nao hardcoded)
5. `.dockerignore` exclui `.git`, `test/`, `docs/`, `.env*`
6. Multi-stage build para imagem minima
7. Graceful shutdown timeout < K8s terminationGracePeriodSeconds
