# Roteiro de Teste Ponta a Ponta — FIAP Cloud Games (TC3, Grupo 17)

**Modelo:** plataforma no **Kubernetes** + **Function serverless por fora** (`func start`).
Serve para **(1)** subir e testar o ambiente e **(2)** gravar a demonstração. Cada passo tem
o comando e **“✅ o que você deve ver”** — siga de cima para baixo.

> **Por que híbrido?** A imagem do Azure Functions é amd64-only e não roda em k8s no Apple
> Silicon. Rodando a Function **como processo** (`func start`) o problema some, e o k8s cobre
> o que o PDF exige em manifestos (observabilidade + persistência). A Function fora do cluster
> ilustra bem o “serverless que sobe só quando há evento”.

---

## 0. Visão geral

```
              (host / sua máquina)                         (Kubernetes - namespace fcg)
  ┌───────────────────────────────┐            ┌──────────────────────────────────────────┐
  │  func start                    │  AMQP      │  RabbitMQ (+ponte)  Mongo  Redis  Postgres │
  │  fcg-notifications-function    │◄──────────►│  Users API   Catalog API   Payments API    │
  │  (OnUserCreated/OnPayment...)  │  Mongo     │  Prometheus  Grafana                        │
  │  azurite (storage)             │◄──────────►│                                            │
  └───────────────────────────────┘ port-fwd   └──────────────────────────────────────────┘
```

**O que será demonstrado (mapeado ao PDF):**
| Item do PDF | Onde aparece no roteiro |
|---|---|
| Serverless acionada por fila, sem container 24/7 | Partes 2 e 4 (logs do `func`) |
| Persistência poliglota (Mongo) + Cache (Redis) | Parte 5 (processed_events, chaves Redis) |
| Observabilidade (Prometheus/Grafana em k8s) | Parte 5 (Grafana) |
| Idempotência / entrega confiável | Parte 5 (dedup por MessageId) |
| API Gateway (Kong) |

---

## 1. Pré-requisitos (instalar uma vez)

| Ferramenta | Para quê | Como conferir |
|---|---|---|
| Docker Desktop **com Kubernetes habilitado** | cluster + azurite | `kubectl config current-context` → `docker-desktop` |
| `kubectl` | aplicar manifestos | `kubectl version --client` |
| .NET 8 SDK | buildar APIs e a Function | `dotnet --version` → 8.x |
| Azure Functions Core Tools v4 (`func`) | rodar a Function | `func --version` → 4.x |
| (imagem) azurite | storage da Function | `docker pull mcr.microsoft.com/azure-storage/azurite` |

> **Mac (Homebrew) — faça uma vez:** o binário in-process do `func` vem sem permissão de
> execução. Rode:
> ```bash
> chmod +x /opt/homebrew/Cellar/azure-functions-core-tools@4/*/in-proc8/func
> chmod +x /opt/homebrew/Cellar/azure-functions-core-tools@4/*/in-proc6/func
> ```

**Layout esperado dos repositórios** (todos irmãos, na mesma pasta):
```
Apps_Grupo17/
  fcg-users-api  fcg-catalog-api  fcg-payments-api  fcg-contracts
  fcg-notifications-function   fcg-orchestration
```

**Branches deste trabalho** (enquanto não houver merge):
```bash
git -C fcg-orchestration            checkout feature/persistencia-mongo-redis-raul
git -C fcg-notifications-function   checkout feature/persistencia-mongo-redis-raul
```

> Abra **4 terminais** (vou chamá-los de **T1..T4**). Ajuda muito na hora da demo.

---

## 2. PARTE A — Subir a plataforma no Kubernetes  *(terminal T1)*

Todos os comandos a partir da pasta `Apps_Grupo17/`.

### A.1 — Buildar as imagens das 3 APIs
```bash
docker build -t fcg-users-api:latest     fcg-users-api
docker build -t fcg-catalog-api:latest   fcg-catalog-api
docker build -t fcg-payments-api:latest  fcg-payments-api
```
✅ Três builds terminam com `naming to docker.io/library/fcg-*:latest done`.

### A.2 — Publicar as imagens no registry local com **TAG ÚNICA** (à prova de cache)
> **Por quê:** o Kubernetes do Docker Desktop mantém um cache próprio (containerd). Com a tag
> `:latest`, ele **reaproveita a imagem antiga** e ignora o seu novo build — foi o que causou
> `/metrics` 404 e código desatualizado. A solução robusta é **registry local + uma tag nova a
> cada subida**.
```bash
# registry local (recria se já existir — evita "container parado" da sessão anterior; 5000=AirPlay no mac)
docker rm -f fcg-registry 2>/dev/null; docker run -d -p 5001:5000 --name fcg-registry registry:2
curl -s http://localhost:5001/v2/_catalog >/dev/null && echo "registry OK" || echo "registry FALHOU (docker no ar?)"

# tag única + push das 3 imagens
export TAG=build-$(date +%s)
for i in users catalog payments; do
  docker tag  fcg-$i-api:latest localhost:5001/fcg-$i-api:$TAG
  docker push localhost:5001/fcg-$i-api:$TAG
done
echo "TAG=$TAG"     # anote — usado na A.3.1
```
> **kind / minikube:** não precisa de registry — recarregue a imagem (isso já atualiza o conteúdo):
> `for i in users catalog payments; do kind load docker-image fcg-$i-api:latest; done`
> (ou `minikube image load fcg-$i-api:latest`). Nesse caso **pule a A.3.1**.

### A.3 — Aplicar os manifestos (⚠️ **pule o `07`** — é a NotificationsAPI 24/7 antiga)
```bash
cd fcg-orchestration
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/03-secrets.yaml
kubectl apply -f k8s/01-postgres.yaml -f k8s/02-rabbitmq.yaml -f k8s/08-redis.yaml -f k8s/09-mongo.yaml
kubectl apply -f k8s/04-users-api.yaml -f k8s/05-catalog-api.yaml -f k8s/06-payments-api.yaml
kubectl apply -f k8s/08-prometheus.yaml -f k8s/09-grafana.yaml
kubectl apply -f k8s/10-kong-postgres.yaml    
kubectl apply -f k8s/11-kong-migrations.yaml  
kubectl apply -f k8s/12-kong.yaml             
kubectl apply -f k8s/13-kong-config.yaml

```
> **NÃO** rode `kubectl apply -f k8s/07-notifications-api.yaml`. Quem faz notificações agora é a Function.

### A.3.1 — (Docker Desktop) Apontar as APIs para a imagem fresca do registry
```bash
for i in users catalog payments; do
  kubectl set image -n fcg deploy/$i-api $i-api=localhost:5001/fcg-$i-api:$TAG
done
kubectl rollout status -n fcg deploy/users-api
kubectl rollout status -n fcg deploy/catalog-api
kubectl rollout status -n fcg deploy/payments-api
```
✅ Garante que os pods rodam o **build atual** (com o `/metrics` do Prometheus e a idempotência do Payments). *(kind/minikube: pule — a imagem já foi recarregada na A.2.)*

### A.4 — Aguardar tudo ficar pronto
```bash
kubectl get pods -n fcg -w
```
✅ Todos os pods em `Running` e `READY 1/1` (Ctrl+C para sair do watch). Mongo e RabbitMQ
sobem sem CrashLoop (probes já corrigidas).

### A.5 — Conferir que a ponte de filas subiu sozinha
```bash
kubectl port-forward -n fcg svc/rabbitmq-service 15672:15672   # deixe rodando neste T1
# em qualquer terminal:
curl -s -u guest:guest http://localhost:15672/api/queues/%2F | grep -o 'notifications-[a-z-]*'
```
✅ Aparecem `notifications-user-created` e `notifications-payment-processed` (carregadas pelo ConfigMap, sem passo manual).

### A.6 — Subir o API Gateway (Kong) - Se não executados no passo anterior  *(T1)*
Kong **DB-backed** (Postgres próprio + migrations). Suba os 4 manifestos na ordem:
```bash
kubectl apply -f k8s/10-kong-postgres.yaml    # banco do Kong
kubectl apply -f k8s/11-kong-migrations.yaml  # Job: bootstrap do schema do Kong
kubectl apply -f k8s/12-kong.yaml             # proxy :8000 / admin :8001 / NodePort 30080
kubectl apply -f k8s/13-kong-config.yaml      # Job: cria services, rotas, consumer e plugin jwt
```
Aguarde (o pod do Kong pode reiniciar até a migração terminar — é normal):
```bash
kubectl wait -n fcg --for=condition=complete job/kong-migrations --timeout=180s
kubectl rollout status -n fcg deploy/kong --timeout=180s
kubectl wait -n fcg --for=condition=complete job/kong-config --timeout=180s
```
✅ `kong-migrations` e `kong-config` = **Completed**; pod `kong` **1/1**.

**Validar a configuração (Admin API):**
```bash
kubectl port-forward -n fcg svc/kong-admin-service 8001:8001 &   # deixe rodando
curl -s http://localhost:8001/services  | python3 -c "import sys,json;print('services :',[s['name'] for s in json.load(sys.stdin)['data']])"
curl -s http://localhost:8001/routes    | python3 -c "import sys,json;print('routes   :',[r['name'] for r in json.load(sys.stdin)['data']])"
curl -s http://localhost:8001/consumers | python3 -c "import sys,json;print('consumers:',[c['username'] for c in json.load(sys.stdin)['data']])"
curl -s http://localhost:8001/plugins   | python3 -c "import sys,json;d=json.load(sys.stdin)['data'];print('plugins  :',[p['name'] for p in d])"
```
✅ services `users-api`/`catalog-api`; rotas `auth-route`,`users-route`,`catalog-*`; consumer `fcg-user`; plugin `jwt`.

**GUI — Kong Manager (visual, bom para a demo):** a GUI (porta 8002) chama o Admin API (8001) **pelo
navegador**, então precisa das **duas** portas encaminhadas. O `12-kong.yaml` já define
`KONG_ADMIN_GUI_URL`/`KONG_ADMIN_GUI_API_URL` (habilita a GUI + CORS).
```bash
kubectl port-forward -n fcg svc/kong-admin-service 8001:8001 &   # a GUI chama aqui
kubectl port-forward -n fcg deploy/kong            8002:8002 &   # a GUI em si
# abra http://localhost:8002 → Services / Routes / Consumers / Plugins (lista tudo)
```
> **"could not be retrieved" na GUI, ou `curl` vazio na 8001?** É port-forward, não é o Kong: a config
> está no banco (a Job `kong-config` cria tudo). Faça `pkill -f "kubectl port-forward"`, garanta o Kong
> Ready (`kubectl rollout status -n fcg deploy/kong`) e suba **os dois** forwards (8001 **e** 8002) de novo.
> *(O `12-kong.yaml` tem um initContainer que evita restart do Kong no boot, mantendo os forwards estáveis.)*
> Alternativa 100% confiável (sem GUI): Admin API no navegador — `http://localhost:8001/services`, `/routes`, `/plugins` (JSON).

---

## 3. PARTE B — Rodar a Function por fora  *(terminais T2 e T3)*

### B.1 — Storage (azurite) — *T2*
```bash
docker run -d --name fcg-azurite -p 10000:10000 -p 10001:10001 -p 10002:10002 \
  mcr.microsoft.com/azure-storage/azurite
```
✅ `docker ps` mostra `fcg-azurite` Up.

### B.2 — Túneis para o cluster (RabbitMQ + Mongo + Redis + Grafana + Prometheus + KONG + SWAGGER) — *T2* (deixe rodando)
```bash
kubectl port-forward -n fcg svc/rabbitmq-service 5672:5672 &
kubectl port-forward -n fcg svc/mongo-service    27017:27017 &
kubectl port-forward -n fcg svc/redis-service 6379:6379 &
kubectl port-forward -n fcg svc/grafana    3000:3000 &
kubectl port-forward -n fcg svc/prometheus 9090:9090 &
kubectl port-forward -n fcg svc/kong-proxy-service 8000:80 &
kubectl port-forward -n fcg svc/kong-admin-service 8001:8001 &
kubectl port-forward -n fcg deploy/kong 8002:8002 &
kubectl port-forward -n fcg svc/users-api-service   8081:80 &
kubectl port-forward -n fcg svc/catalog-api-service 8082:80 &
```
✅ `Forwarding from 127.0.0.1:5672` e `...:27017`.

### B.3 — Configurar e iniciar a Function — *T3*
```bash
cd fcg-notifications-function/src/FCG.Notifications.Functions # ENTRAR NO REPO DA FUNCTION
cp local.settings.json.example local.settings.json     # já aponta p/ localhost
func start
```
✅ Você vê:
```
Functions:
  Health: [GET] http://localhost:7071/api/health
  OnPaymentProcessed: rabbitMQTrigger
  OnUserCreated: rabbitMQTrigger
```
**Deixe este terminal (T3) visível na gravação — é aqui que os “e-mails” aparecem.**

---

## 4. PARTE C — Exercitar as funcionalidades  *(terminal T4)*

### C.0 — Expor as APIs
```bash
kubectl port-forward -n fcg svc/users-api-service   8081:80 &
kubectl port-forward -n fcg svc/catalog-api-service 8082:80 &
```
Swagger: Users → http://localhost:8081/swagger · Catalog → http://localhost:8082/swagger
(o Swagger está habilitado; o ambiente é `Docker`, não `Production`.)

> Você pode fazer tudo pelo **Swagger** (mais visual) ou pelos `curl` abaixo.

### C.1 — Cadastrar um usuário  → dispara **boas-vindas**
```bash
curl -s -X POST http://localhost:8081/api/usuarios/registrar \
  -H "content-type: application/json" \
  -d '{"nome":"Raul Demo","email":"raul.demo@fcg.com","senha":"Senha@123"}'
```
✅ HTTP **201**. **No T3 (func)** aparece:
`[NOTIFICAÇÃO] ✉ E-mail de boas-vindas ENVIADO | Para: raul.demo@fcg.com | ...`

### C.2 — Criar um admin, logar e cadastrar um jogo (para ter o que comprar)
```bash
# admin
curl -s -X POST http://localhost:8081/api/usuarios/administradores \
  -H "content-type: application/json" \
  -d '{"nome":"Admin Demo","email":"admin@fcg.com","senha":"Senha@123"}'

ou utilize o usuario login já disponível - {"email":"admin@fcg.com","senha":"TrocarEm@2026"}

# login admin → copie o accessToken
curl -s -X POST http://localhost:8081/api/auth/login \
  -H "content-type: application/json" \
  -d '{"email":"admin@fcg.com","senha":"Senha@123"}'
ADMIN=<cole_o_accessToken_aqui>

# criar jogo → copie o id retornado
curl -s -X POST http://localhost:8082/api/jogos \
  -H "authorization: Bearer $ADMIN" -H "content-type: application/json" \
  -d '{"titulo":"Elden Ring","descricao":"RPG de acao","genero":"RPG","preco":199.90}'
JOGO=<cole_o_id_do_jogo>
```
✅ Jogo criado (**201**) com `id` e `preco: 199.90`.

### C.3 — Logar como o usuário e **comprar** → dispara pagamento + **confirmação**
```bash
# login do usuário do C.1
curl -s -X POST http://localhost:8081/api/auth/login \
  -H "content-type: application/json" \
  -d '{"email":"raul.demo@fcg.com","senha":"Senha@123"}'
USER=<cole_o_accessToken_do_usuario>

# comprar o jogo
curl -s -X POST http://localhost:8082/api/biblioteca/aquisicoes \
  -H "authorization: Bearer $USER" -H "content-type: application/json" \
  -d "{\"jogoId\":\"$JOGO\"}"
```
✅ HTTP **202** (pagamento é assíncrono). Em ~1–2s, **no T3 (func)** aparece:
`[NOTIFICAÇÃO] ✉ E-mail de confirmação de compra ENVIADO | ... | Jogo: Elden Ring | Preço: R$ 199.90 | ...`
> Isso prova o fluxo completo: Catalog → OrderPlaced → Payments → PaymentProcessed → **Function**.

---

## 4½. PARTE C-GW — Fluxo pela **entrada única** (API Gateway / Kong)  *(T4)*
Mesmas funcionalidades da Parte C, porém **tudo passando pelo Kong** (roteamento + validação de JWT).
```bash
kubectl port-forward -n fcg svc/kong-proxy-service 8000:80 &   # use port-forward (no Docker Desktop o NodePort :30080 costuma NÃO responder)
```

### GW.1 — Rota pública funciona sem token
```bash
curl -s -o /dev/null -w "GET /api/jogos SEM token -> %{http_code}\n" http://localhost:8000/api/jogos
```
✅ **200** (rota pública roteada para a CatalogAPI).

### GW.2 — Rota protegida SEM token é barrada NO GATEWAY
```bash
curl -s -o /dev/null -w "POST /api/jogos SEM token -> %{http_code}\n" -X POST http://localhost:8000/api/jogos
```
✅ **401** — o Kong bloqueia **antes** de chegar na API (prova a validação de JWT no gateway).

### GW.3 — Login do admin (rota pública) → token
> **Não copie o token na mão** — o JWT é longo, quebra de linha no terminal e corrompe a assinatura
> (erro `Bad token; invalid signature` no Kong). **Capture numa variável:**
```bash
ADMIN=$(curl -s -X POST http://localhost:8000/api/auth/login -H "content-type: application/json" \
  -d '{"email":"admin@fcg.com","senha":"TrocarEm@2026"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['accessToken'])")
echo "len do token: ${#ADMIN}"   # ~500 = ok
```
> O admin é criado automaticamente (seed) na subida da UsersAPI.

### GW.4 — Cadastrar um usuário comum (logado como admin) → dispara **boas-vindas**
> A rota `/api/usuarios` exige JWT no gateway; por isso o cadastro vai com o token do admin. O
> endpoint cria um usuário **comum** a partir do corpo (ignora quem chamou). Use um **e-mail novo**
> (se repetir, a API responde **409 "E-mail já cadastrado"** — o que também prova que passou pelo gateway).
```bash
curl -s -X POST http://localhost:8000/api/usuarios/registrar \
  -H "authorization: Bearer $ADMIN" -H "content-type: application/json" \
  -d '{"nome":"Raul teste","email":"raul_teste@fcg.com","senha":"Senha@123"}' -w "\nHTTP %{http_code}\n"
```
✅ **201** e, no **T3 (func)**: `E-mail de boas-vindas ENVIADO | Para: raul.demo@fcg.com …`.

### GW.5 — Login do usuário comum, criar jogo (admin) e comprar (user) — tudo pelo gateway
```bash
# token do usuário comum (capturado em variável)
USER=$(curl -s -X POST http://localhost:8000/api/auth/login -H "content-type: application/json" \
  -d '{"email":"raul_teste@fcg.com","senha":"Senha@123"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['accessToken'])")

# criar jogo (token do ADMIN) e capturar o id do jogo
JOGO=$(curl -s -X POST http://localhost:8000/api/jogos -H "authorization: Bearer $ADMIN" -H "content-type: application/json" \
  -d '{"titulo":"EFC 27","descricao":"FUTEBOL","genero":"RPG","preco":599.90}' | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

# comprar (token do USER)
curl -s -X POST http://localhost:8000/api/biblioteca/aquisicoes -H "authorization: Bearer $USER" -H "content-type: application/json" \
  -d "{\"jogoId\":\"$JOGO\"}" -w "\nHTTP %{http_code}\n"
```
✅ Compra **202** e, no T3: `E-mail de confirmação de compra ENVIADO … Preço: R$ 199.90`.

> **Nota (defeito conhecido):** `POST /api/usuarios/registrar` **sem** token retorna **401** no gateway —
> o auto-cadastro público está bloqueado pela rota `/api/usuarios` com JWT. O contorno é cadastrar
> logado como admin (GW.4). Correção definitiva: expor `POST /api/usuarios/registrar` como rota pública no Kong.

---

## 5. PARTE D — Visualizar os resultados

### D.1 — Logs da Function (serverless acionada) — **T3**
Já visível: os dois `[NOTIFICAÇÃO] …`. É o “exibir os logs da função” do PDF.

### D.2 — Persistência NoSQL / idempotência (Mongo)
```bash
kubectl exec -n fcg deploy/mongo -- mongosh --quiet fcg_notifications \
  --eval 'db.processed_events.find({},{_id:0,consumer:1,messageId:1,businessKey:1}).toArray()'
```
✅ 2 documentos: `UserCreatedEvent` e `PaymentProcessedEvent`, com o índice único garantindo a dedup.

### D.3 — Idempotência ao vivo (reentrega não duplica)
Reenvie o mesmo cadastro do C.1 (mesmo request):
```bash
curl -s -X POST http://localhost:8081/api/usuarios/registrar \
  -H "content-type: application/json" \
  -d '{"nome":"Raul Demo","email":"raul.demo@fcg.com","senha":"Senha@123"}'
```
✅ Retorna **409** (e-mail já existe) — e **nenhum novo e-mail** é logado. Para provar a dedup no
nível de evento (mesmo `MessageId`), veja o teste unitário `dedup` ou reenvie um evento pela
API de management do RabbitMQ; a contagem em `processed_events` **não** aumenta.

### D.4 — Cache (Redis)
```bash
# gere cache consultando o catálogo 2x (a 2ª vem do Redis)
curl -s http://localhost:8082/api/jogos >/dev/null; curl -s http://localhost:8082/api/jogos >/dev/null
kubectl exec -n fcg deploy/redis -- redis-cli KEYS 'catalog:*'
```
✅ Aparecem chaves `catalog:jogos:*` (cache-aside do Catalog).

### D.5 — Observabilidade (Grafana/Prometheus)
```bash
kubectl port-forward -n fcg svc/grafana    3000:3000 &
kubectl port-forward -n fcg svc/prometheus 9090:9090 &
```
- **Prometheus** — http://localhost:9090 → **Status → Target health**: `users-api` e `catalog-api`
  devem estar **UP** (verde). Se aparecerem **DOWN com 404** em `/metrics`, a imagem está velha →
  refaça **A.2 / A.3.1** (tag única + `set image`).
- **Grafana** — http://localhost:3000 (`admin` / `<GRAFANA_ADMIN_PASSWORD>`) → menu **Dashboards** →
  **"FCG - Observabilidade das APIs"** (ou direto em http://localhost:3000/d/fcg-apis).
  O datasource Prometheus e o dashboard **já sobem provisionados** pelo manifesto `09-grafana.yaml`
  — nada para configurar/importar na mão.
✅ Painéis de **requisições/s por status**, **por serviço**, **latência p95** e **erros 5xx**.
Gere tráfego (Parte C), ajuste o range para "Last 15 minutes" e refresh 5s para ver os gráficos se moverem.
> ⚠️ **Não** use **Drilldown → Traces**: aquilo precisa de um datasource de traces (Tempo), que
> **não** faz parte da Opção A (métricas). As métricas ficam em **Dashboards** / **Explore**.

---

## 6. PARTE E — Encerrar / limpar
```bash
# parar a Function: Ctrl+C no T3
# parar os port-forwards: Ctrl+C / kill %1 %2 ... nos terminais
docker rm -f fcg-azurite
kubectl delete namespace fcg          # remove toda a plataforma
```

---

## 7. Troubleshooting rápido
| Sintoma | Causa / correção |
|---|---|
| Pod da API `ImagePullBackOff` | imagem não visível ao cluster → refaça a **A.2** (registry / `kind load` / `minikube image load`) |
| `func start`: *Failed to start inproc8 host … Permission denied* | Mac/Homebrew → `chmod +x .../in-proc8/func` (§1) |
| Function não recebe eventos | faltou o **port-forward do rabbitmq (5672)** ou a ponte não subiu (cheque A.5) |
| Function sobe mas erro de storage | azurite não está no ar (B.1) |
| Swagger 404 | ambiente em `Production` desliga o Swagger; nos manifestos é `Docker` (ok). Se editou, mantenha `ASPNETCORE_ENVIRONMENT=Docker` |
| Mongo/RabbitMQ CrashLoop | use os manifestos desta branch (probes já corrigidas: mongo `tcpSocket`+`Recreate`) |

---

