# Itaú Transaction Authorizer

API de autorização de transações financeiras (crédito/débito) desenvolvida para o desafio técnico Itaú Unibanco.

## Funcionalidades

- Consumo da fila SQS `conta-bancaria-criada` (LocalStack) para cadastro de contas com saldo inicial zero
- Autorização de transações via `POST /transactions/{transactionId}`
- Idempotência por `transactionId`
- Lock pessimista para consistência de saldo em concorrência
- Observabilidade: logs JSON, métricas Prometheus, health checks
- Resiliência: retry com exponential backoff + full jitter, circuit breaker (Resilience4j)

## Stack

- Java 21, Spring Boot 3.4, PostgreSQL 16, Flyway
- AWS SDK v2 (SQS), LocalStack
- Arquitetura Hexagonal (Ports & Adapters)

## Pré-requisitos

- Docker e Docker Compose
- (Opcional) AWS CLI para inspecionar a fila SQS

## Execução local

### Opção 1 — Stack completa (recomendada)

```bash
cd itau-transaction-authorizer
docker compose up --build
```

Serviços:

| Serviço | Porta | Descrição |
|---------|-------|-----------|
| authorizer-app | 8080 | API REST |
| postgres | 5432 | Banco de dados |
| localstack | 4566 | SQS local |
| message-generator | — | Gera 100k contas (executa uma vez) |

Aguarde `message-generator exited with code 0` — a fila estará populada.

### Opção 2 — Apenas infra + app (sem gerador)

Se a fila já foi populada:

```bash
docker compose up postgres localstack authorizer-app --build
```

### Verificar fila SQS

```bash
export AWS_DEFAULT_REGION=sa-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
aws --endpoint-url=http://localhost:4566 --region sa-east-1 sqs receive-message \
  --queue-url http://localhost:4566/000000000000/conta-bancaria-criada \
  --max-number-of-messages 10
```

## API

### Autorizar transação

```http
POST /transactions/{transactionId}
Content-Type: application/json

{
  "accountId": "5b19c8b6-0cc4-4c72-a989-0c2ee15fa975",
  "type": "CREDIT",
  "amount": { "value": 97.07, "currency": "BRL" }
}
```

**Resposta (200):**

```json
{
  "transaction": {
    "id": "8e8ae808-b154-48b5-9f3e-553935cc4543",
    "type": "CREDIT",
    "amount": { "value": 97.07, "currency": "BRL" },
    "status": "SUCCEEDED",
    "timestamp": "2025-07-08T15:57:55-03:00"
  },
  "account": {
    "id": "5b19c8b6-0cc4-4c72-a989-0c2ee15fa975",
    "balance": { "amount": 183.12, "currency": "BRL" }
  }
}
```

### Endpoints auxiliares

- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI: http://localhost:8080/api-docs
- Health: http://localhost:8080/actuator/health
- Métricas: http://localhost:8080/actuator/prometheus

## Semântica HTTP

| Cenário | HTTP | `transaction.status` |
|---------|------|------------------------|
| Aprovado | 200 | SUCCEEDED |
| Conta inexistente | 200 | FAILED |
| Saldo insuficiente | 200 | FAILED |
| Payload inválido | 400 | — |

## Testes

```bash
# Unitários (exclui integração)
docker run --rm -v $(pwd):/app -w /app maven:3.9.9-eclipse-temurin-21-alpine mvn test

# Integração (requer Docker para Testcontainers)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/app -w /app \
  maven:3.9.9-eclipse-temurin-21-alpine mvn test -Dtest='!*Integration*' -DexcludedGroups= -Dgroups=integration
```

## Documentação adicional

- [Arquitetura — diagramas Mermaid](docs/architecture/README.md) — C4, deploy local/nuvem, fluxogramas
- [ADRs](docs/adr/) — Decisões arquiteturais
- [Deploy Cloud](docs/architecture/cloud-deployment.md)
- [Pipeline CI/CD](docs/ci-cd/pipeline-canary.md)
- [Coleção Postman](postman/itau-authorizer.postman_collection.json)

## Future Improvements

- DLQ dedicada para mensagens SQS com falha permanente
- Event sourcing com Kafka para auditoria completa
- Cache read-only de saldo (Redis) para consultas de alta frequência
- Sharding por `account_id` para escala horizontal extrema
