# 💳 pocket-ledger

> API de carteira digital com transferências atômicas, idempotência por chave e controle de concorrência em saldo.

![Java](https://img.shields.io/badge/Java-21-orange?style=flat-square)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?style=flat-square)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=flat-square)
![Redis](https://img.shields.io/badge/Redis-7-DC382D?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)

---

## Sobre o projeto

O **pocket-ledger** é uma API REST de carteira digital onde usuários podem depositar saldo, transferir para outras contas e consultar extrato com histórico imutável.

O objetivo do projeto não é simular um banco completo — é demonstrar como resolver os problemas reais que qualquer sistema financeiro enfrenta:

- O que acontece quando dois usuários tentam gastar o mesmo saldo ao mesmo tempo?
- Como garantir que um retry de rede não processe a mesma transferência duas vezes?
- Como garantir que um registro financeiro nunca seja alterado após criado?

Cada decisão técnica deste projeto tem uma motivação de negócio clara, documentada abaixo.

---

## Como rodar

**Pré-requisitos:** Docker e Docker Compose instalados.

```bash
# Clone o repositório
git clone https://github.com/seu-usuario/pocket-ledger.git
cd pocket-ledger

# Suba toda a infraestrutura
docker-compose up -d

# A API estará disponível em:
# http://localhost:8080
```

O `docker-compose.yml` sobe automaticamente:
- PostgreSQL na porta `5432`
- Redis na porta `6379`
- Grafana na porta `3000` (usuário: `admin`, senha: `admin`)
- Prometheus na porta `9090`

As migrations do Flyway rodam automaticamente na inicialização da aplicação.

---

## Endpoints

### Carteira

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/wallets/me` | Retorna saldo atual e dados da carteira |
| `POST` | `/wallets/me/deposit` | Deposita valor (requer `Idempotency-Key` no header) |
| `POST` | `/wallets/me/transfer` | Transfere valor para outra carteira |
| `GET` | `/wallets/me/statement` | Extrato paginado com filtro por período e tipo |

### Transações

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/transactions/{id}` | Detalhe de uma transação específica |
| `GET` | `/transactions/{id}/idempotency` | Consulta se uma operação já foi processada |

Todos os endpoints exigem autenticação via JWT no header `Authorization: Bearer <token>`.

### Exemplos de request

**Depósito:**
```http
POST /wallets/me/deposit
Authorization: Bearer <token>
Idempotency-Key: dep-2024-01-15-001
Content-Type: application/json

{
  "amount": 250.00
}
```

**Transferência:**
```http
POST /wallets/me/transfer
Authorization: Bearer <token>
Idempotency-Key: trf-2024-01-15-002
Content-Type: application/json

{
  "toWalletId": "018e1234-5678-7abc-9def-0123456789ab",
  "amount": 100.00
}
```

---

## Arquitetura

### Diagrama de classes

```
┌─────────────────────────┐         ┌──────────────────────────────────┐
│         Wallet          │         │           Transaction             │
├─────────────────────────┤         ├──────────────────────────────────┤
│ + id: UUID (v7)         │  1   *  │ + id: UUID (v7)                  │
│ + userId: UUID          │─────────│ + fromWalletId: UUID (nullable)  │
│ + balance: NUMERIC(19,4)│         │ + toWalletId: UUID               │
│ + version: Long         │         │ + amount: NUMERIC(19,4)          │
│ + createdAt: Timestamp  │         │ + type: TransactionType          │
├─────────────────────────┤         │ + status: TransactionStatus      │
│ + deposit(amount): void │         │ + idempotencyKey: String         │
│ + debit(amount): void   │         │ + createdAt: Timestamp           │
│ + credit(amount): void  │         └──────────────┬───────────────────┘
└─────────────────────────┘                        │ 1
                                                   │
                                       ┌───────────▼──────────────────┐
                                       │        IdempotencyKey         │
                                       ├──────────────────────────────┤
                                       │ + keyHash: String (PK)        │
                                       │ + userId: UUID                │
                                       │ + responseStatus: Integer     │
                                       │ + responseBody: JSONB         │
                                       │ + expiresAt: Timestamp        │
                                       └──────────────────────────────┘

 TransactionType        TransactionStatus
 ──────────────         ─────────────────
 DEPOSIT                PENDING
 TRANSFER               COMPLETED
                        FAILED
```

### Diagrama de casos de uso

```
                    ┌──────────────────────────────────────────────────────┐
                    │                   pocket-ledger                       │
                    │                                                        │
                    │   ╭────────────────────────╮                          │
                    │   │    Depositar saldo      │───«include»──▶ Validar idempotência
   ┌──────────┐     │   ╰────────────────────────╯                          │
   │          │     │                                                        │
   │ Usuário  │────▶│   ╭────────────────────────╮                          │
   │autenticado     │   │   Transferir valor      │───«include»──▶ Validar saldo suficiente
   │          │     │   ╰────────────────────────╯                          │
   └──────────┘     │                                                        │
                    │   ╭────────────────────────╮                          │
                    │   │   Consultar saldo       │                          │
                    │   ╰────────────────────────╯                          │
                    │                                                        │
                    │   ╭────────────────────────╮                          │
                    │   │     Ver extrato         │                          │
                    │   ╰────────────────────────╯                          │
                    │                                                        │
                    │   ╭────────────────────────╮                          │
                    │   │ Ver detalhe de transação│                          │
                    │   ╰────────────────────────╯                          │
                    └──────────────────────────────────────────────────────┘
```

### Fluxo de transferência

```
Cliente                  API                    Banco              Redis
  │                       │                       │                  │
  │── POST /transfer ────▶│                       │                  │
  │   Idempotency-Key: X  │                       │                  │
  │                       │── consulta chave X ──────────────────────▶│
  │                       │◀─ não existe ─────────────────────────────│
  │                       │                       │                  │
  │                       │── BEGIN TRANSACTION ─▶│                  │
  │                       │── SELECT wallet (lock)▶│                  │
  │                       │── valida saldo        │                  │
  │                       │── UPDATE debit ───────▶│                  │
  │                       │── UPDATE credit ──────▶│                  │
  │                       │── INSERT transaction ─▶│                  │
  │                       │── COMMIT ─────────────▶│                  │
  │                       │                       │                  │
  │                       │── salva chave X + resposta ──────────────▶│
  │◀── 200 OK ────────────│                       │                  │
  │                       │                       │                  │
  │   (retry por timeout) │                       │                  │
  │── POST /transfer ────▶│                       │                  │
  │   Idempotency-Key: X  │                       │                  │
  │                       │── consulta chave X ──────────────────────▶│
  │                       │◀─ já existe, retorna resposta original ───│
  │◀── 200 OK (cached) ───│                       │                  │
```

---

## Decisões técnicas

Cada decisão aqui tem uma motivação concreta. Este é o tipo de explicação que você dará em entrevistas técnicas.

### Por que UUID v7 em vez de v4?

UUID v4 é completamente aleatório. Quando inserido em um índice B-tree do PostgreSQL, cada novo registro vai para uma posição aleatória na árvore, causando **page splits** e fragmentação do índice conforme a tabela cresce.

UUID v7 é **time-ordered**: os primeiros bits são um timestamp Unix em milissegundos. Registros novos são sempre maiores que os anteriores, então inserções no índice são quase sempre no final — sem fragmentação.

O resultado prático: tabelas de transações financeiras crescem muito. UUID v7 mantém a performance de inserção estável independente do volume.

### Por que NUMERIC(19,4) em vez de FLOAT ou DOUBLE?

Tipos de ponto flutuante (`float`, `double`) usam representação binária que não consegue representar com precisão alguns decimais. O exemplo clássico:

```java
System.out.println(0.1 + 0.2); // 0.30000000000000004
```

Em sistemas financeiros, esse erro se acumula. Usar `NUMERIC(19,4)` no PostgreSQL e `BigDecimal` no Java garante **aritmética exata**. Nunca use `float` para dinheiro.

### Por que Optimistic Lock com @Version?

Duas requisições simultâneas tentando debitar o mesmo saldo são um problema real. Existem duas abordagens:

- **Pessimistic lock** (`SELECT FOR UPDATE`): bloqueia a linha no banco até o commit. Simples, mas cria contenção — outros acessos ficam esperando.
- **Optimistic lock** (`@Version`): não bloqueia nada. Cada transação lê o saldo e o número de versão. Na hora do commit, se a versão mudou desde a leitura, lança `OptimisticLockException` e o chamador faz retry.

Para carteiras com poucos conflitos simultâneos (o caso mais comum), o optimistic lock é mais performático. Para contas com altíssimo volume concorrente, pessimistic lock pode ser mais adequado — e essa é a conversa que vale ter em entrevista.

### Por que Idempotency Key?

Redes são não-confiáveis. Um cliente pode enviar uma requisição de transferência, o servidor processar com sucesso, mas a resposta se perder antes de chegar ao cliente. O cliente então faz retry — e sem idempotência, a transferência é processada duas vezes.

O padrão `Idempotency-Key` resolve isso:

1. O cliente gera uma chave única por operação (ex: UUID) e envia no header
2. O servidor verifica se já processou essa chave
3. Se sim, retorna o resultado original sem processar de novo
4. Se não, processa e salva o resultado atrelado à chave

A chave expira em 24 horas. Esse é exatamente o comportamento da API de pagamentos da Stripe.

### Por que extrato imutável (append-only)?

Nenhum registro de transação é atualizado ou deletado após criado. O status muda de `PENDING` para `COMPLETED` ou `FAILED`, mas o registro em si permanece.

Isso garante uma **trilha de auditoria completa**: você sempre pode reconstruir o saldo de qualquer carteira em qualquer ponto no tempo a partir do histórico de transações. É um requisito de compliance em sistemas financeiros reais.

---

## Stack

| Categoria | Tecnologia | Por quê |
|-----------|-----------|---------|
| Runtime | Java 21 + Spring Boot 3.x | LTS atual, virtual threads disponíveis |
| Persistência | PostgreSQL 16 + Spring Data JPA | ACID completo, NUMERIC nativo |
| Migrations | Flyway | Schema como código, versionado no Git |
| Cache/Idempotência | Redis 7 | TTL nativo, sub-milissegundo |
| Resiliência | Resilience4j | Retry com backoff configurável |
| Observabilidade | Micrometer + Prometheus + Grafana | Stack padrão de produção |
| Testes | JUnit 5 + Testcontainers | Infraestrutura real nos testes |
| Containerização | Docker Compose | Um comando para subir tudo |

---

## Estrutura do projeto

```
pocket-ledger/
├── src/
│   ├── main/
│   │   ├── java/com/pocketledger/
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   └── RedisConfig.java
│   │   │   ├── domain/
│   │   │   │   ├── wallet/
│   │   │   │   │   ├── Wallet.java
│   │   │   │   │   ├── WalletRepository.java
│   │   │   │   │   ├── WalletService.java
│   │   │   │   │   └── WalletController.java
│   │   │   │   ├── transaction/
│   │   │   │   │   ├── Transaction.java
│   │   │   │   │   ├── TransactionType.java
│   │   │   │   │   ├── TransactionStatus.java
│   │   │   │   │   ├── TransactionRepository.java
│   │   │   │   │   ├── TransactionService.java
│   │   │   │   │   └── TransactionController.java
│   │   │   │   └── idempotency/
│   │   │   │       ├── IdempotencyKey.java
│   │   │   │       ├── IdempotencyKeyRepository.java
│   │   │   │       └── IdempotencyService.java
│   │   │   ├── exception/
│   │   │   │   ├── InsufficientFundsException.java
│   │   │   │   ├── WalletNotFoundException.java
│   │   │   │   └── GlobalExceptionHandler.java
│   │   │   └── shared/
│   │   │       ├── UuidV7Generator.java
│   │   │       └── IdempotencyFilter.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │           ├── V1__create_wallets.sql
│   │           ├── V2__create_transactions.sql
│   │           └── V3__create_idempotency_keys.sql
│   └── test/
│       ├── java/com/pocketledger/
│       │   ├── wallet/
│       │   │   ├── WalletServiceTest.java
│       │   │   └── WalletControllerTest.java
│       │   ├── transaction/
│       │   │   ├── TransferServiceTest.java
│       │   │   ├── TransferConcurrencyTest.java   ← teste das 10 threads
│       │   │   └── IdempotencyTest.java
│       │   └── BaseIntegrationTest.java           ← Testcontainers base
└── docker-compose.yml
```

---

## Testes

```bash
# Rodar todos os testes
./mvnw test

# Rodar apenas testes de integração
./mvnw test -Dgroups=integration

# Rodar o teste de concorrência
./mvnw test -Dtest=TransferConcurrencyTest
```

O teste de concorrência `TransferConcurrencyTest` dispara 10 threads simultâneas tentando debitar o mesmo saldo. Ao final, valida que:
- Apenas as transferências com saldo suficiente completaram
- O saldo final é exatamente o esperado
- Nenhuma transação ficou em estado inconsistente

---

## Observabilidade

Com o projeto rodando, acesse:

- **Grafana**: http://localhost:3000 — dashboard com métricas de volume, latência e taxa de erro
- **Prometheus**: http://localhost:9090 — métricas brutas
- **Actuator**: http://localhost:8080/actuator — health, info e métricas

Métricas expostas:

| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `pocket_ledger.deposit.total` | Counter | Total de depósitos por status |
| `pocket_ledger.transfer.total` | Counter | Total de transferências por status |
| `pocket_ledger.transfer.amount` | DistributionSummary | Volume financeiro movimentado |
| `pocket_ledger.idempotency.hit` | Counter | Requisições servidas do cache de idempotência |
| `http.server.requests` | Timer | Latência por endpoint (p50, p95, p99) |

---

## Variáveis de ambiente

| Variável | Padrão | Descrição |
|----------|--------|-----------|
| `DB_HOST` | `localhost` | Host do PostgreSQL |
| `DB_PORT` | `5432` | Porta do PostgreSQL |
| `DB_NAME` | `pocket_ledger` | Nome do banco |
| `DB_USER` | `postgres` | Usuário do banco |
| `DB_PASSWORD` | `postgres` | Senha do banco |
| `REDIS_HOST` | `localhost` | Host do Redis |
| `REDIS_PORT` | `6379` | Porta do Redis |
| `JWT_SECRET` | — | Chave para validação do JWT (obrigatória) |

---
