# FCG Orchestration

Repositório de orquestração da plataforma **Fiap Cloud Games (FCG)** — Fases 3 e 4.

Contém o `docker-compose.yml` unificado (ambiente local) e todos os manifests Kubernetes
consolidados (`k8s/`), que hoje rodam em produção no **Azure Kubernetes Service (AKS)**.

## Produção no Azure (TC4)

| Item | Recurso |
|---|---|
| Kubernetes gerenciado | AKS `aks-fcg-tc4` (plano Free), 2 nós `Standard_D2as_v4`, região East US, resource group `rg-fcg-tc4` |
| Registry privado | Azure Container Registry `acrfcgtc4b` (`acrfcgtc4b.azurecr.io`) — o AKS puxa as imagens com identidade gerenciada (AcrPull), sem senha |
| Segredos | Azure Key Vault `kv-fcg-tc4b` → driver CSI + Workload Identity (ver [Segredos](#segredos)) |
| Exposição externa | Kong Gateway como Service `LoadBalancer` (Azure Load Balancer). Todo o resto é `ClusterIP` (interno) |
| CI/CD | GitHub Actions em `fcg-users-api` e `fcg-catalog-api`: build → testes → imagem com tag do commit → scan (Trivy) → push no ACR → Rolling Update no AKS |
| Busca avançada | Elasticsearch 8.15.3 no AKS (`k8s/14-elasticsearch.yaml`), usado pelo CatalogAPI em `GET /api/jogos/search` |
| Observabilidade | Prometheus + Grafana (Grafana exposto por `LoadBalancer`) |

Endereços públicos (mudam se o cluster for desligado e religado — `az aks stop/start`):

```bash
kubectl get svc -n fcg kong-proxy-service grafana     # coluna EXTERNAL-IP
```

### Mapa das exigências da Fase 4

| Exigência | Onde está |
|---|---|
| 1. Cluster gerenciado, registry privado e Load Balancer | AKS + ACR + `k8s/12-kong.yaml` (`kong-proxy-service` tipo `LoadBalancer`) |
| 2. NoSQL (MongoDB) e Cache (Redis) | `k8s/09-mongo.yaml`, `k8s/08-redis.yaml`; uso no código descrito em [Persistência poliglota](#persistência-poliglota) |
| 3. Busca avançada com Elasticsearch | `k8s/14-elasticsearch.yaml` + indexação/busca no `fcg-catalog-api` ([Busca avançada](#busca-avançada-elasticsearch)) |
| 4. CI/CD com Rolling Update | `.github/workflows/ci.yml` dos repos `fcg-users-api` e `fcg-catalog-api` ([CI/CD](#cicd-github-actions--acr--aks)) |
| 5. Zero hardcoded credentials | `k8s/03-keyvault-secrets.yaml` (Key Vault + CSI) — nenhum valor sensível nos manifests |

## Arquitetura

```
                    ┌─────────────────────────────────────────────┐
                    │               FCG Platform                  │
                    │                                             │
  Cliente HTTP ────►│  Kong Gateway (JWT) ──► UsersAPI / CatalogAPI│
                    │                                             │
                    │  UsersAPI  ─── publica ──► UserCreatedEvent │
                    │  CatalogAPI ── publica ──► OrderPlacedEvent │
                    │         │                                   │
                    │         │  ◄── consome ── PaymentProcessed  │
                    │         │                 Event             │
                    │         │                     ▲             │
                    │     PaymentsAPI ───────────────┘            │
                    │         (consome OrderPlaced, publica       │
                    │          PaymentProcessed)                  │
                    │                                             │
                    │     NotificationsAPI                        │
                    │         (consome UserCreated + Payment      │
                    │          Processed — logs e-mail simulado)  │
                    │                                             │
                    │  PostgreSQL    (identidade · loja ·         │
                    │                 biblioteca)                 │
                    │  MongoDB       (fcg_catalog · fcg_payments ·│
                    │                 fcg_notifications)          │
                    │  Redis         (cache-aside do catálogo)    │
                    │  Elasticsearch (índice "jogos" — busca)     │
                    │  RabbitMQ      (eventos entre serviços)     │
                    └─────────────────────────────────────────────┘
```

### Persistência poliglota

A plataforma combina quatro tecnologias de armazenamento, cada uma resolvendo um
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
  troca de latência menor e menos carga no banco relacional. Inserir/editar um jogo
  invalida as chaves.
- **Elasticsearch** — motor de busca **independente** do banco relacional, com busca
  tolerante a erros de digitação e ordenação por relevância (ver abaixo).

## Busca avançada (Elasticsearch)

- **Onde roda:** `k8s/14-elasticsearch.yaml` — Elasticsearch 8.15.3 single-node no AKS, com
  disco persistente (PVC 5 Gi), autenticação ligada (usuário `elastic`, senha vinda do Key Vault
  `elastic-password`) e Service `ClusterIP` (não exposto à internet).
- **Quem usa:** o CatalogAPI, via variáveis `Elasticsearch__Url`, `Elasticsearch__Username`,
  `Elasticsearch__IndexName` (ConfigMap do `k8s/05-catalog-api.yaml`) e `Elasticsearch__Password`
  (Secret `platform-secrets`).
- **Sincronização:** criar/editar jogo → indexa; desativar → remove do índice; reativar → indexa.
  No startup, o CatalogAPI cria o índice `jogos` (se não existir) e reconcilia com o Postgres.
- **Endpoint:** `GET /api/jogos/search?q=<termo>` (público, via Kong) — `multi_match` com
  `fuzziness: AUTO` (pesos título 3, gênero 2, descrição 1), resultados ordenados por `score`.
  Detalhes no README do `fcg-catalog-api`.

```bash
curl -s "http://<IP-do-Kong>/api/jogos/search?q=aventrua" | jq '.[] | {titulo, score}'
```

## Fluxo de eventos

```
UsersAPI          ──[UserCreatedEvent]──────────────────► NotificationsAPI
CatalogAPI        ──[OrderPlacedEvent]──────────────────► PaymentsAPI
PaymentsAPI       ──[PaymentProcessedEvent]─────────────► CatalogAPI + NotificationsAPI
```

## Segredos

O repositório é público, então **nenhuma credencial é versionada**.

### AKS (TC4): Azure Key Vault + CSI Secret Store

As credenciais de produção vivem **só** no Azure Key Vault `kv-fcg-tc4b`. O cluster
as busca em tempo de execução, sem senha nenhuma guardada:

```text
Key Vault kv-fcg-tc4b ──► driver CSI (addon do AKS) ──► Secret "platform-secrets" ──► Deployments
        ▲                       │
        └── identidade id-fcg-kv (Workload Identity, federada com a SA fcg:fcg-sa)
```

- `k8s/03-keyvault-secrets.yaml` cria a ServiceAccount `fcg-sa` e o
  `SecretProviderClass` `fcg-keyvault`, que lista os **12 segredos** do cofre e, via
  `secretObjects`, **gera** o Secret `platform-secrets` com as chaves usadas pelos
  Deployments (`Jwt__Secret`, `ConnectionStrings__Postgres`, `ELASTIC_PASSWORD`…).
- Segredos no cofre: `postgres-user`, `postgres-password`, `postgres-conn`, `jwt-secret`,
  `rabbitmq-password`, `mongo-conn`, `seed-admin-senha`, `kong-postgres-user`,
  `kong-postgres-password`, `kong-postgres-db`, `grafana-admin-password`, `elastic-password`.
- As 4 APIs montam o volume CSI em `/mnt/secrets-store`; o Secret existe enquanto
  ao menos um pod montar o volume.
- O arquivo só contém identificadores (tenant, client ID da identidade, nome do
  cofre), que não dão acesso a nada sozinhos.
- Prova de origem: `kubectl get secret platform-secrets -n fcg --show-labels`
  mostra `secrets-store.csi.k8s.io/managed=true`.
- Para trocar um valor: grave uma nova versão no cofre e reinicie os pods que o usam
  (`kubectl rollout restart deploy/<api> -n fcg`).
- **Novo segredo:** além de criá-lo no cofre e listá-lo no `03`, é preciso regenerar o
  Secret (o driver não acrescenta chaves num Secret existente):
  ```bash
  kubectl apply -f k8s/03-keyvault-secrets.yaml
  kubectl delete secret platform-secrets -n fcg
  kubectl rollout restart deploy/payments-api -n fcg     # um pod que monta o volume recria o Secret
  ```

### Ambiente local (kind/Docker Desktop)

O modelo com placeholders fica **fora** de `k8s/` (em `exemplos/`), para que um
`kubectl apply -f k8s/` nunca aplique placeholders por cima do Secret real. O
arquivo real `k8s/03-secrets.yaml` está no `.gitignore`.

> ⚠️ `k8s/` está adaptado ao AKS (ACR, `managed-csi`, Key Vault).
> Fora do Azure o `03-keyvault-secrets.yaml` e o volume CSI das APIs não funcionam.

```
Copy-Item exemplos\03-secrets.example.yaml k8s\03-secrets.yaml
# edite k8s/03-secrets.yaml e troque cada <placeholder> por um valor seu
kubectl apply -f k8s/03-secrets.yaml
```

> Se o Postgres local já foi criado com outra senha, mantenha o seu arquivo atual:
> o Postgres só lê a senha na primeira criação do banco.

## Deploy no AKS (produção)

Pré-requisitos no Azure: AKS com OIDC issuer, Workload Identity e o addon
`azure-keyvault-secrets-provider`; ACR anexado ao AKS (`--attach-acr`); Key Vault com os 12
segredos; identidade `id-fcg-kv` com *Key Vault Secrets User* e credencial federada para
`system:serviceaccount:fcg:fcg-sa`.

```bash
az aks get-credentials -g rg-fcg-tc4 -n aks-fcg-tc4 --overwrite-existing

# 1. Namespace e segredos (Key Vault)
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/03-keyvault-secrets.yaml

# 2. Todo o restante (infra, APIs, Elasticsearch, Kong, observabilidade)
kubectl apply -f k8s/
kubectl get pods -n fcg -w
```

**Normal nos primeiros minutos:** pods de infraestrutura em `CreateContainerConfigError` até o
`platform-secrets` nascer (ele é criado quando o primeiro pod de API monta o volume CSI) e APIs
reiniciando até Postgres/RabbitMQ ficarem prontos — tudo se resolve sozinho. Os jobs
`kong-migrations` e `kong-config` terminam como `Completed`.

### Imagens e tags (quem é dono da tag)

Os manifests `04`–`07` apontam para `acrfcgtc4b.azurecr.io/<serviço>:v1` (versão inicial).
Para `users-api` e `catalog-api` **o pipeline é o dono da tag**: cada push na `main` publica
`:<sha do commit>` e faz `kubectl set image`. Por isso, ao reaplicar `04`/`05` em produção,
preserve a imagem em uso para não voltar à `v1`:

```bash
IMG=$(kubectl get deploy catalog-api -n fcg -o jsonpath='{.spec.template.spec.containers[0].image}')
sed "s#acrfcgtc4b.azurecr.io/fcg-catalog-api:v1#${IMG}#" k8s/05-catalog-api.yaml | kubectl apply -f -
```

`payments-api` e `notifications-api` não têm pipeline e usam `:v1`.

## CI/CD (GitHub Actions → ACR → AKS)

Workflows em `fcg-users-api/.github/workflows/ci.yml` e `fcg-catalog-api/.github/workflows/ci.yml`:

1. **build-and-test** — restore, build e testes automatizados (`dotnet test`).
2. **Publicar no ACR e atualizar o AKS** (só na `main`):
   - login no Azure por **OIDC** (identidade `id-fcg-github`, sem senha guardada no GitHub);
   - build da imagem com tag = SHA do commit;
   - **scan de vulnerabilidades (Trivy)**;
   - push no ACR;
   - `kubectl set image` + `rollout status` — **Rolling Update** (`maxUnavailable: 0`, `maxSurge: 1`), sem downtime;
   - `rollout undo` automático se o rollout falhar.

Configuração no GitHub (por repositório): Secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`,
`AZURE_SUBSCRIPTION_ID`; Variables `ACR_NAME`, `ACR_LOGIN_SERVER`, `AKS_RESOURCE_GROUP`,
`AKS_CLUSTER_NAME`, `K8S_NAMESPACE`.

> As credenciais federadas da `id-fcg-github` precisam casar com o *subject* que o GitHub envia.
> Hoje ele inclui os IDs da organização e do repositório
> (`repo:Fiap-Grupo-17@<org-id>/<repo>@<repo-id>:ref:refs/heads/main`); se o login falhar com
> `AADSTS700213`, compare o subject do erro com
> `az identity federated-credential list -g rg-fcg-tc4 --identity-name id-fcg-github -o table`.

## API Gateway — Kong

A plataforma utiliza o **Kong Gateway** (DB mode) como único ponto de entrada das APIs.

- `k8s/10-kong-postgres.yaml` — banco PostgreSQL utilizado pelo Kong.
- `k8s/11-kong-migrations.yaml` — migrations/bootstrap do banco do Kong.
- `k8s/12-kong.yaml` — Deployment e Services do Kong (`kong-proxy-service` `LoadBalancer`; Admin API `ClusterIP`).
- `k8s/13-kong-config.yaml` — Job que cria Services, Routes, plugin JWT e o consumer.

### Rotas publicadas

| Rota | Métodos | Caminho | JWT no Kong |
|---|---|---|---|
| `auth-route` | todos | `/api/auth` | não |
| `users-registrar-route` | POST | `/api/usuarios/registrar` | não |
| `users-route` | todos | `/api/usuarios` | **sim** |
| `catalog-jogos-get-route` | GET | `/api/jogos` (inclui `/api/jogos/search`) | não |
| `catalog-jogos-post-route` | POST, PUT, PATCH | `/api/jogos` (criar, editar, ativar/desativar) | **sim** (+ perfil Admin na API) |
| `catalog-aquisicoes-route` | POST | `/api/biblioteca/aquisicoes` | **sim** |
| `catalog-biblioteca-route` | GET | `/api/biblioteca` | **sim** |

`/swagger`, `/health` e `/metrics` **não** são publicados no Kong (respondem 404 pelo gateway).

### Testar pelo Kong

```bash
KONG=$(kubectl get svc kong-proxy-service -n fcg -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -i http://$KONG/api/jogos                                         # público
curl -i -H "Authorization: Bearer SEU_TOKEN_JWT" http://$KONG/api/usuarios   # protegido
```

Quando a requisição passa pelo Kong, a resposta traz cabeçalhos como `Via: kong/3.4.x`,
`X-Kong-Proxy-Latency` e `X-Kong-Upstream-Latency` — eles separam erro de roteamento de erro
interno da aplicação.

### Administração do Kong

A Admin API é interna (`ClusterIP`). Para consultar:

```bash
kubectl port-forward -n fcg svc/kong-admin-service 18001:8001
curl http://localhost:18001/routes
```

> Não exiba `kubectl logs job/kong-config`: o log do job contém a credencial JWT do consumer.

### Fluxo simplificado

```text
Swagger / Postman / curl
          |
          | http://<IP público do Kong>
          v
      Kong Gateway (valida JWT)
          |
          +----> users-api-service:80   ----> UsersAPI   ----> PostgreSQL
          |
          +----> catalog-api-service:80 ----> CatalogAPI ----> PostgreSQL · Redis · MongoDB · Elasticsearch
```

## Observabilidade

Optamos pela **Opção A: stack de código aberto (Prometheus + Grafana)**.

- `UsersAPI` e `CatalogAPI` foram instrumentadas com o pacote `prometheus-net.AspNetCore`,
  expondo métricas HTTP no endpoint `/metrics` (interno).
- O Prometheus (`k8s/08-prometheus.yaml`) faz scrape desses endpoints a cada 15s, configurado
  via ConfigMap apontando para os Services internos (`users-api-service:80` e
  `catalog-api-service:80`).
- O Grafana (`k8s/09-grafana.yaml`) se conecta ao Prometheus (`http://prometheus:9090`) e já sobe
  com o dashboard provisionado **"FCG - Observabilidade das APIs"**:

| Painel | Query PromQL |
|---|---|
| Requisições/s por status | `sum by (code) (rate(http_requests_received_total[1m]))` |
| Requisições/s por serviço | `sum by (job) (rate(http_requests_received_total[1m]))` |
| Latência p95 por serviço | `histogram_quantile(0.95, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))` |
| Taxa de erros 5xx por serviço | `sum by (job) (rate(http_requests_received_total{code=~"5.."}[1m]))` |

**Acesso no AKS:** `http://<EXTERNAL-IP do Service grafana>` — usuário `admin`, senha no Key Vault
(`grafana-admin-password`). Localmente: `kubectl port-forward svc/grafana 3000:80 -n fcg` (o Service escuta na 80).

## Executar localmente (todos os serviços)

Crie o arquivo `.env` com as senhas do ambiente local (ele não é versionado):

```bash
# PowerShell (Linux/macOS: cp .env.example .env)
Copy-Item .env.example .env
# preencha: POSTGRES_PASSWORD, RABBITMQ_PASSWORD, JWT_SECRET (mínimo 32 caracteres),
# SEED_ADMIN_PASSWORD e GRAFANA_ADMIN_PASSWORD
```

O `docker compose` recusa subir se alguma variável estiver vazia. O login do admin inicial é `admin@fcg.com` com a senha definida em `SEED_ADMIN_PASSWORD`.

> **Pré-requisito**: imagens Docker de cada serviço construídas ou usar `--build`

```bash
# Da raiz deste repositório
docker compose up --build -d

# Acompanhar logs
docker compose logs -f

# Verificar status
docker compose ps
```

> O Elasticsearch não faz parte deste compose; para testar a busca localmente use o
> `docker-compose.yml` do `fcg-catalog-api`, que já inclui o serviço `elasticsearch`.

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
| RabbitMQ Mgmt     | 15672       | http://localhost:15672 (guest/<RABBITMQ_PASSWORD>) |
| Prometheus        | 9090        | http://localhost:9090                |
| Grafana           | 3000        | http://localhost:3000 (admin/<GRAFANA_ADMIN_PASSWORD>) |

### Validar Redis e MongoDB

```bash
# Redis — chaves de cache do catálogo
redis-cli -h localhost -p 6379 KEYS 'catalog:*'

# MongoDB — bancos e coleções de cada serviço
mongosh mongodb://localhost:27017 --eval "db.getSiblingDB('fcg_catalog').getCollectionNames()"
mongosh mongodb://localhost:27017 --eval "db.getSiblingDB('fcg_payments').getCollectionNames()"
mongosh mongodb://localhost:27017 --eval "db.getSiblingDB('fcg_notifications').getCollectionNames()"
```

### Kubernetes local (kind/Docker Desktop)

Com o Secret local criado (ver [Ambiente local](#ambiente-local-kinddocker-desktop)), aplique a
infraestrutura antes das APIs para evitar `CrashLoopBackOff` na primeira tentativa de conexão
(elas se recuperam sozinhas, mas isso agiliza o start):

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/03-secrets.yaml
kubectl apply -f k8s/01-postgres.yaml -f k8s/02-rabbitmq.yaml -f k8s/08-redis.yaml -f k8s/09-mongo.yaml
kubectl apply -f k8s/04-users-api.yaml -f k8s/05-catalog-api.yaml -f k8s/06-payments-api.yaml -f k8s/07-notifications-api.yaml
kubectl apply -f k8s/08-prometheus.yaml -f k8s/09-grafana.yaml
kubectl apply -f k8s/10-kong-postgres.yaml -f k8s/11-kong-migrations.yaml -f k8s/12-kong.yaml -f k8s/13-kong-config.yaml
kubectl get pods -n fcg
```

Em clusters kind o `LoadBalancer`/NodePort nem sempre fica acessível em `localhost`; use port-forward:

```bash
kubectl port-forward -n fcg svc/kong-proxy-service 8000:80 &
kubectl port-forward -n fcg svc/users-api-service   8081:80 &
kubectl port-forward -n fcg svc/catalog-api-service 8082:80 &
kubectl port-forward -n fcg svc/grafana    3000:80 &
kubectl port-forward -n fcg svc/prometheus 9090:9090 &
# Kong (gateway): http://localhost:8000 · UsersAPI: http://localhost:8081/swagger · CatalogAPI: http://localhost:8082/swagger
```

> Os arquivos `08-redis.yaml` e `09-mongo.yaml` mantêm a numeração alta (após as
> APIs) por terem sido adicionados posteriormente à convenção original de
> prefixos, mas devem ser aplicados **antes** dos manifests `04`–`07`.

## Limitações conhecidas

- O RabbitMQ usa o usuário `guest`; o hash da senha está no ConfigMap do `k8s/02-rabbitmq.yaml`
  (a senha em si vem do Key Vault). Próximo passo: usuário dedicado com definições geradas a partir do cofre.
- Os endereços públicos são IPs dinâmicos, sem DNS fixo nem TLS no gateway.

## Repositórios

| Repositório                  | Descrição                                               |
|------------------------------|---------------------------------------------------------|
| `fcg-contracts`              | NuGet de contratos de eventos compartilhados            |
| `fcg-users-api`              | Autenticação, cadastro, JWT (com pipeline CI/CD)        |
| `fcg-catalog-api`            | Catálogo, biblioteca, busca avançada (com pipeline CI/CD) |
| `fcg-payments-api`           | Processamento de pagamentos (event-driven)              |
| `fcg-notifications-api`      | Notificações por e-mail (simulado)                      |
| `fcg-notifications-function` | Notificações por e-mail SERVERLESS (simulado)           |
| `fcg-orchestration`          | Este repositório — compose + k8s                        |

## Material de apoio

- `ordem_execucao_comandos.md` — ordem exata dos comandos para subir o ambiente completo
- `ROTEIRO_TESTE_E2E_HIBRIDO.md` — roteiro de subida do ambiente e usabilidade da aplicação

## Grupo 17 — Pos-Tech FIAP
- Letícia Lopes Ribeiro Vasconcelos
- Marcelo Henrique Cornelis Rei
- Washington Santana dos Santos
- Raul Hentz Rodrigues
