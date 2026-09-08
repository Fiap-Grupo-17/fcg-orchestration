# FCG Orchestration

Repositório de orquestração da plataforma **Fiap Cloud Games (FCG)** — Fase 2.

Contém o `docker-compose.yml` unificado e todos os manifests Kubernetes consolidados.

## Arquitetura

```
                    ┌─────────────────────────────────────────────┐
                    │               FCG Platform                  │
                    │                                             │
  Cliente HTTP ────►│  :8081 UsersAPI  ─── publica ──►            │
                    │                         UserCreatedEvent    │
                    │  :8082 CatalogAPI ─── publica ──►           │
                    │         │               OrderPlacedEvent    │
                    │         │                                   │
                    │         │  ◄── consome ── PaymentProcessed  │
                    │         │                 Event             │
                    │         │                     ▲             │
                    │     :8083 PaymentsAPI ─────────┘            │
                    │         (consome OrderPlaced, publica       │
                    │          PaymentProcessed)                  │
                    │                                             │
                    │     :8084 NotificationsAPI                  │
                    │         (consome UserCreated + Payment      │
                    │          Processed — logs e-mail simulado)  │
                    │                                             │
                    │  PostgreSQL (schemas: identidade · loja     │
                    │              biblioteca)                    │
                    │  MongoDB   :27017 (fcg_catalog · fcg_       │
                    │              payments · fcg_notifications)  │
                    │  Redis     :6379  (cache-aside do catálogo) │
                    │  RabbitMQ  :5672 (AMQP) :15672 (Mgmt UI)    │
                    └─────────────────────────────────────────────┘
```

### Persistência poliglota

A plataforma combina três tecnologias de armazenamento, cada uma resolvendo um
problema diferente:

- **PostgreSQL (EF Core)** — fonte de verdade transacional do CatalogAPI e UsersAPI
  (jogos, pedidos, biblioteca, usuários). Onde a consistência forte é obrigatória.
- **MongoDB** — um único servidor compartilha três bancos lógicos (`fcg_catalog`,
  `fcg_payments`, `fcg_notifications`), usados para dados de alta volumetria/schema
  flexível: read models desnormalizados (catálogo), histórico de pagamentos e
  notificações, e `processed_events` para idempotência dos consumers de eventos.
  Modelar isso em Postgres exigiria migrações constantes e não traria ganho de
  consistência, já que esses dados não participam de transações ACID entre serviços.
- **Redis** — cache-aside para as consultas mais frequentes do catálogo (listagem e
  detalhe de jogo), evitando reconsultar o Postgres a cada requisição em um endpoint
  de alto tráfego. TTL curto (5–10 min) aceita alguma inconsistência temporária em
  troca de latência menor e menos carga no banco relacional.

## Fluxo de eventos

```
UsersAPI          ──[UserCreatedEvent]──────────────────► NotificationsAPI
CatalogAPI        ──[OrderPlacedEvent]──────────────────► PaymentsAPI
PaymentsAPI       ──[PaymentProcessedEvent]─────────────► CatalogAPI + NotificationsAPI
```

## Executar localmente (todos os serviços)

> **Pré-requisito**: imagens Docker de cada serviço construídas ou usar `--build`

```bash
# Da raiz deste repositório
docker compose up --build -d

# Acompanhar logs
docker compose logs -f

# Verificar status
docker compose ps
```

### Portas expostas

| Serviço           | Porta local | Swagger                              |
|-------------------|-------------|--------------------------------------|
| UsersAPI          | 8081        | http://localhost:8081/swagger        |
| CatalogAPI        | 8082        | http://localhost:8082/swagger        |
| PaymentsAPI       | 8083        | health: http://localhost:8083/health |
| NotificationsAPI  | 8084        | health: http://localhost:8084/health |
| PostgreSQL        | 5432        | —                                    |
| MongoDB           | 27017       | —                                    |
| Redis             | 6379        | —                                    |
| RabbitMQ AMQP     | 5672        | —                                    |
| RabbitMQ Mgmt     | 15672       | http://localhost:15672 (guest/guest) |

### Validar Redis e MongoDB

```bash
# Redis — chaves de cache do catálogo
redis-cli -h localhost -p 6379 KEYS 'catalog:*'

# MongoDB — bancos e coleções de cada serviço
mongosh mongodb://localhost:27017 --eval "db.getSiblingDB('fcg_catalog').getCollectionNames()"
mongosh mongodb://localhost:27017 --eval "db.getSiblingDB('fcg_payments').getCollectionNames()"
mongosh mongodb://localhost:27017 --eval "db.getSiblingDB('fcg_notifications').getCollectionNames()"
```

## Deploy no Kubernetes

```bash
# 1. Namespace e segredos primeiro
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/03-secrets.yaml

# 2. Infraestrutura (Postgres, RabbitMQ, Redis, Mongo) — aplicar antes das APIs
#    para evitar que os pods das APIs fiquem em CrashLoopBackOff na primeira
#    tentativa de conexão (eles se recuperam sozinhos, mas isso agiliza o start)
kubectl apply -f k8s/01-postgres.yaml
kubectl apply -f k8s/02-rabbitmq.yaml
kubectl apply -f k8s/08-redis.yaml
kubectl apply -f k8s/09-mongo.yaml

# 3. Microsserviços
kubectl apply -f k8s/04-users-api.yaml
kubectl apply -f k8s/05-catalog-api.yaml
kubectl apply -f k8s/06-payments-api.yaml
kubectl apply -f k8s/07-notifications-api.yaml

# 4. Verificar pods
kubectl get pods -n fcg

# 5. Acessar serviços (NodePort)
# UsersAPI:   http://localhost:30081/swagger
# CatalogAPI: http://localhost:30082/swagger
```

> Os arquivos `08-redis.yaml` e `09-mongo.yaml` mantêm a numeração alta (após as
> APIs) por terem sido adicionados posteriormente à convenção original de
> prefixos, mas devem ser aplicados **antes** dos manifests `04`–`07` conforme a
> ordem acima.

### ⚠️ Antes de aplicar em produção

1. **Altere o `Jwt__Secret`** em `k8s/03-secrets.yaml` (mínimo 32 caracteres aleatórios)
2. **Altere as senhas** do PostgreSQL e RabbitMQ
3. **Build e push** das imagens para um registry (ex: Docker Hub, ECR, GCR):
   ```bash
   docker build -t seu-registry/fcg-users-api:1.0.0 ../fcg-users-api
   docker push seu-registry/fcg-users-api:1.0.0
   # ... repita para os demais serviços
   ```
4. **Atualize** `image:` nos arquivos `04` a `07` com o caminho do registry
5. Considere usar **Kubernetes Secrets** gerenciados (Vault, AWS Secrets Manager)

## Repositórios

| Repositório            | Descrição                                     |
|------------------------|-----------------------------------------------|
| `fcg-contracts`        | NuGet de contratos de eventos compartilhados  |
| `fcg-users-api`        | Autenticação, cadastro, JWT                   |
| `fcg-catalog-api`      | Catálogo de jogos + Biblioteca                |
| `fcg-payments-api`     | Processamento de pagamentos (event-driven)    |
| `fcg-notifications-api`| Notificações por e-mail (simulado)            |
| `fcg-orchestration`    | Este repositório — compose + k8s              |

## Grupo 17 — Pos-Tech FIAP
- Letícia Lopes Ribeiro Vasconcelos
- Marcelo Henrique Cornelis Rei
- Vinícius Calixto Real
