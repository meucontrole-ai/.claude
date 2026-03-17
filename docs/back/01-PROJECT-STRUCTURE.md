# Project Structure

## Quando Ativar
- Criar novos arquivos ou pacotes
- Decidir onde colocar novo codigo
- Refatorar organizacao de arquivos

---

## Como Pensar

Este projeto segue Hexagonal Architecture (Ports & Adapters). A regra fundamental:
**dependencias apontam para dentro** — adapters dependem de ports, ports dependem de domain, domain nao depende de nada.

Nao pensamos em "camadas MVC". Pensamos em: "esse codigo e regra de negocio (domain), orquestracao (application), contrato (ports) ou implementacao tecnica (adapters)?"

Se voce nao sabe onde colocar algo, pergunte: "se eu trocar o banco de dados, esse codigo muda?" Se sim, e adapter. Se nao, e domain ou application.

---

## Estrutura Real do Projeto

```
internal/
  domain/                          # Regras de negocio puras
    payment/                       # Agregado principal
      payment.go                   # Aggregate root + construtor + mutacoes
      status.go                    # Maquina de estados + validTransitions
      command.go                   # Commands (input DTOs com primitivos)
      result.go                    # PaymentResult (output DTO)
      events.go                    # Eventos de dominio
      method.go                    # Union discriminada: Card | Pix | Boleto
      method_type.go               # MethodType enum
      card.go, pix.go, boleto.go   # Value objects de cada metodo
      billing.go                   # BillingDetails + BillingAddress
      attempt.go                   # Tentativa ao provider
      capture_method.go            # CaptureMethod enum
      confirmation_method.go       # ConfirmationMethod enum
      provider_result.go           # Resultado do provider
    shared/                        # Tipos compartilhados entre agregados
      errors.go                    # DomainError + codigos
      events.go                    # DomainEvent interface + AggregateRoot
      money.go                     # Money value object (int64 centavos)
      identity.go                  # GenerateID() via uuid
      metadata.go                  # Metadata map[string]string
    company/                       # Agregado Company
    customer/                      # Agregado Customer
    fraud/                         # Assessment de fraude
    refund/                        # Agregado Refund
    saga/                          # Commands e events do saga
    split/                         # Regras de split
    transaction/                   # Agregado Transaction
    webhook/                       # Input e record de webhook

  ports/                           # APENAS interfaces, nada mais
    inbound/                       # Contratos de entrada (use cases)
      create_payment.go            # CreatePaymentUseCase interface
      authorize_payment.go         # AuthorizePaymentUseCase interface
      capture_payment.go           # ...
      cancel_payment.go
      refund_payment.go
      process_webhook.go
      validate_payment.go
    outbound/                      # Contratos de saida (infra)
      payment_repository.go        # PaymentRepository interface
      payment_provider.go          # PaymentProvider interface
      event_publisher.go           # EventPublisher interface
      idempotency_store.go         # IdempotencyStore interface
      company_repository.go
      customer_repository.go
      fraud_repository.go
      refund_repository.go
      transaction_repository.go
      saga_event_publisher.go
      routing_strategy.go
      webhook_event_handler.go
      webhook_event_store.go
      webhook_signature_validator.go

  application/                     # Orquestracao (use cases agrupados por entidade)
    delivery/                      # Use cases de delivery
      assign_driver/usecase.go     # package assigndriver
      cancel/usecase.go            # package canceldelivery
      confirm/usecase.go           # package confirmdelivery
      confirm_pickup/usecase.go    # package confirmpickup
      create/usecase.go            # package createdelivery
      query/usecase.go             # package deliveryquery
      report_problem/usecase.go    # package reportproblem
      unassign_driver/usecase.go   # package unassigndriver
      update_status/usecase.go     # package updatestatus
    driver/                        # Use cases de driver
      auth/                        # Sub-grupo: autenticacao
        confirm_password_reset/    # package confirmpasswordreset
        request_password_reset/    # package requestpasswordreset
        set_password/              # package setpassword
        validate_reset_token/      # package validateresettoken
      avatar/                      # Sub-grupo: avatar
        delete/                    # package deleteavatar
        get/                       # package getavatar
        upload/                    # package uploadavatar
      position/                    # Sub-grupo: posicao
        publish/                   # package publishposition
        query/                     # package positionquery
      create/usecase.go            # package createdriver
      deactivate/usecase.go        # package deactivatedriver
      go_offline/usecase.go        # package gooffline
      go_online/usecase.go         # package goonline
      query/usecase.go             # package driverquery
      suspend/usecase.go           # package suspenddriver
      take_break/usecase.go        # package takebreak
      unsuspend/usecase.go         # package unsuspenddriver
      update/usecase.go            # package updatedriver
      update_location/usecase.go   # package updatelocation
    driver_merchant/               # Use cases de driver-merchant
      activate/usecase.go          # package activatedrivermerchant
      associate/usecase.go         # package associatedrivermerchant
      deactivate/usecase.go        # package deactivatedrivermerchant
      delete/usecase.go            # package deletedrivermerchant
      query/usecase.go             # package drivermerchantquery
    merchant/                      # Use cases de merchant
      settings/                    # Sub-grupo: configuracoes
        get/                       # package getmerchantsettings
        update/                    # package updatemerchantsettings
      activate/usecase.go          # package activatemerchant
      create/usecase.go            # package createmerchant
      deactivate/usecase.go        # package deactivatemerchant
      query/usecase.go             # package merchantquery
      update/usecase.go            # package updatemerchant
    shared/                        # Utilitarios compartilhados

  adapters/                        # Implementacoes tecnicas
    grpc/
      handler/                     # package handler (service + handlers)
        service.go                 # PaymentService struct
        payment_handler.go         # CreatePayment, Authorize, Capture...
        company_handler.go
        customer_handler.go
      mapper/                      # package mapper (proto <-> domain)
        mapper.go                  # ToPaymentResponsePB, ToCompanyPB...
      interceptor.go               # Interceptors gRPC
    webhook/                       # package webhook (Fiber HTTP)
      handlers.go
      matera_http.go
      matera_payment_handler.go
      middleware.go
      payload_parser.go
      event_types.go
    messaging/                     # Consumidores de mensagens
      handler/                     # package handler (validation_handler)
      dto/                         # DTOs de mensageria
      memory/                      # Consumer in-memory (dev/test)
      sqs/                         # Consumer SQS (producao)
    repository/
      postgres/                    # package postgres
        model.go                   # PaymentModel (GORM struct com tags)
        mapper.go                  # ToModel / ToDomain
        payment_repository.go      # PostgresPaymentRepository
        database.go                # Conexao e migracao
        tracing.go                 # Spans de DB
      memory/                      # Repos in-memory para testes
    events/
      eventbridge/                 # Publisher EventBridge (producao)
      memory/                      # Publisher in-memory (dev/test)
      noop/                        # Publisher noop (service_core)
      dto/                         # Payloads de eventos
    providers/
      registry.go                  # Registry de providers
      router.go                    # SmartRouter
      resilient_provider.go        # ResilientProvider wrapper
      matera/                      # Implementacao Matera
      stripe/                      # Implementacao Stripe
      c6/                          # Implementacao C6
    polling/                       # Workers de polling

  bootstrap/                       # Composition root
    service_core.go                # Wiring do servico gRPC
    service_webhook.go             # Wiring do servico webhook
    service_saga.go                # Wiring do servico saga
    providers.go                   # BuildProviders
    database.go                    # NewDatabase

  config/
    config.go                      # Config via env vars

cmd/
  service_core/main.go             # Binario gRPC
  service_webhook/main.go          # Binario HTTP webhook
  service_saga/main.go             # Binario SQS consumer
  saga_cli/main.go                 # CLI para testes

pkg/
  logger/                          # Zerolog wrapper
  telemetry/                       # OpenTelemetry setup
```

---

## Regra de Dependencia (Obrigatoria)

```
adapters -> ports -> domain
adapters -> application -> ports -> domain
bootstrap -> adapters + application + ports + domain

domain NAO importa ports
domain NAO importa application
domain NAO importa adapters
ports NAO importa adapters
application NAO importa adapters
```

---

## Onde Colocar Cada Coisa

| O que? | Onde? | Exemplo real |
|---|---|---|
| Aggregate root | `domain/<aggregate>/` | `domain/payment/payment.go` |
| Value object | `domain/<aggregate>/` ou `domain/shared/` | `domain/shared/money.go` |
| Domain event | `domain/<aggregate>/events.go` | `domain/payment/events.go` |
| Command (input) | `domain/<aggregate>/command.go` | `domain/payment/command.go` |
| Result (output) | `domain/<aggregate>/result.go` | `domain/payment/result.go` |
| Use case interface | `ports/inbound/` | `ports/inbound/create_payment.go` |
| Repository interface | `ports/outbound/` | `ports/outbound/payment_repository.go` |
| Use case impl | `application/<entity>/<use_case>/usecase.go` | `application/delivery/create/usecase.go` |
| GORM model | `adapters/repository/postgres/model.go` | `PaymentModel` |
| Repo impl | `adapters/repository/postgres/` | `payment_repository.go` |
| Domain<->Model mapper | `adapters/repository/postgres/mapper.go` | `ToModel()`, `ToDomain()` |
| gRPC handler | `adapters/grpc/handler/` | `payment_handler.go` |
| Proto<->Domain mapper | `adapters/grpc/mapper/` | `mapper.go` |
| Provider impl | `adapters/providers/<name>/` | `adapters/providers/matera/` |
| Bootstrap/wiring | `bootstrap/` | `service_core.go` |

---

## Convencoes de Nomenclatura

- **Packages**: snake_case simples (`createpayment`, nao `create_payment`)
  - Excecao: diretorios usam underscore (`create_payment/`), mas package name junta (`createpayment`)
- **Arquivos**: snake_case (`payment_handler.go`, `usecase.go`)
- **Structs**: PascalCase descritivo (`PaymentService`, `UseCase`, `PostgresPaymentRepository`)
- **Interfaces**: sem prefixo "I" (`PaymentRepository`, nao `IPaymentRepository`)
- **Construtores**: `New` + tipo (`NewPayment`, `NewUseCase`, `NewPostgresPaymentRepository`)
- **Metodos de mutacao**: verbo de negocio (`Authorize`, `Capture`, `Cancel`, nao `SetStatus`)
- **Helpers privados**: camelCase (`buildMethod`, `transitionTo`, `touch`)

---

## Regras

1. Domain NUNCA importa pacotes externos (exceto stdlib e uuid)
2. Ports contem APENAS interfaces — zero structs, zero DTOs, zero implementacao
3. Commands e Results vivem em `domain/`, NAO em `ports/`
4. Use cases agrupados por entidade em `application/<entity>/<use_case>/` (sub-grupos permitidos: auth, avatar, position, settings)
5. Cada adapter tem seu proprio package (nao misture grpc com webhook)
6. gRPC handler fica em `adapters/grpc/handler/` (package `handler`), mapper em `adapters/grpc/mapper/` (package `mapper`)
7. GORM models ficam APENAS em `adapters/repository/postgres/` — nunca no domain
8. Bootstrap e o unico lugar que conhece todos os packages
