# FCG Orchestration

Repositório de orquestração da plataforma **Fiap Cloud Games (FCG)** — Fase 3.

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
                    │  RabbitMQ  :5672 (AMQP) :15672 (Mgmt UI)    │
                    └─────────────────────────────────────────────┘
```

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
| RabbitMQ AMQP     | 5672        | —                                    |
| RabbitMQ Mgmt     | 15672       | http://localhost:15672 (guest/guest) |
— 
| Prometheus        | 9090        | http://localhost:9090                |
— 
| Grafana           | 3000        | http://localhost:3000 (admin/admin1234) |

## Deploy no Kubernetes

```bash
# 1. Aplicar na ordem dos prefixos numéricos
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-postgres.yaml
kubectl apply -f k8s/02-rabbitmq.yaml
kubectl apply -f k8s/03-secrets.yaml
kubectl apply -f k8s/04-users-api.yaml
kubectl apply -f k8s/05-catalog-api.yaml
kubectl apply -f k8s/06-payments-api.yaml
kubectl apply -f k8s/07-notifications-api.yaml
kubectl apply -f k8s/08-prometheus.yaml
kubectl apply -f k8s/09-grafana.yaml

# 2. Verificar pods
kubectl get pods -n fcg

# 3. Acessar serviços
# Em clusters criados via kind (como o do Docker Desktop mais recente), o NodePort
# não fica acessível diretamente em localhost. Use kubectl port-forward:
kubectl port-forward svc/users-api-service 8081:80 -n fcg
kubectl port-forward svc/catalog-api-service 8082:80 -n fcg
kubectl port-forward svc/prometheus 9090:9090 -n fcg
kubectl port-forward svc/grafana 3000:3000 -n fcg

# UsersAPI:   http://localhost:8081/swagger
# CatalogAPI: http://localhost:8082/swagger
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000 (admin/admin1234)

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


## Observabilidade

Optamos pela **Opção A: stack de código aberto (Prometheus + Grafana)**.

### Como funciona

- `UsersAPI` e `CatalogAPI` foram instrumentadas com o pacote `prometheus-net.AspNetCore`,
  expondo métricas HTTP automaticamente no endpoint `/metrics`.
- O Prometheus (`k8s/08-prometheus.yaml`) faz scrape desses endpoints a cada 15s, configurado
  via ConfigMap apontando para os Services internos (`users-api-service:80` e
  `catalog-api-service:80`).
- O Grafana (`k8s/09-grafana.yaml`) se conecta ao Prometheus (`http://prometheus:9090`,
  nome do Service interno) como fonte de dados.

### Dashboard

O dashboard "FCG — Observabilidade" contém 3 painéis:

| Painel | Query PromQL |
|---|---|
| Latência (p95) | `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))` |
| Requisições por status code | `sum(rate(http_requests_received_total[5m])) by (job, code)` |
| Taxa de Erro % | `(sum(rate(http_requests_received_total{code=~"[45].."}[5m])) or vector(0)) / sum(rate(http_requests_received_total[5m])) * 100` |

### Subir a stack

```bash
kubectl apply -f k8s/08-prometheus.yaml
kubectl apply -f k8s/09-grafana.yaml
```

### Acessar

```bash
kubectl port-forward svc/prometheus 9090:9090 -n fcg
kubectl port-forward svc/grafana 3000:3000 -n fcg
```

- Prometheus: http://localhost:9090 (verificar em Status > Targets se `users-api` e
  `catalog-api` aparecem como UP)
- Grafana: http://localhost:3000 — usuário `admin`, senha `admin1234`

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
