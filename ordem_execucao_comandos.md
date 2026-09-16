# Ordem de Execução — Subir o ambiente FCG (TC3, Grupo 17)

Checklist linear, de cima para baixo. Cada bloco tem **✅ o que deve aparecer**.
Modelo: **plataforma no Kubernetes + Function serverless por fora** (`func start`).

> **Shell:** os comandos assumem **bash** (macOS/Linux, ou **Git Bash / WSL2** no Windows).
> No Windows, rode em **Git Bash/WSL** (funciona igual) — veja as notas de PowerShell no fim.
> **Só no Mac (Homebrew):** rode uma vez `chmod +x /opt/homebrew/Cellar/azure-functions-core-tools@4/*/in-proc8/func` (no Windows/Linux **pule**).

Abra **5 terminais** (T1..T5). Todos os `docker build`/`kubectl` a partir da pasta `Apps_Grupo17/`.

---

## 0) Pré-checagem (T1)
```bash
kubectl config current-context      # -> docker-desktop (ou kind/minikube)
docker version >/dev/null && echo "docker ok"
func --version                      # -> 4.x
dotnet --version                    # -> 8.x
```

## 1) Branches corretas (T1)
```bash
git -C fcg-orchestration          checkout feature/persistencia-mongo-redis-raul
git -C fcg-notifications-function checkout feature/persistencia-mongo-redis-raul
```

## 2) Build das 3 imagens (T1)
```bash
docker build -t fcg-users-api:latest     fcg-users-api
docker build -t fcg-catalog-api:latest   fcg-catalog-api
docker build -t fcg-payments-api:latest  fcg-payments-api
```
✅ 3 builds terminam sem erro.

## 3) Registry local + TAG ÚNICA + push (T1)  ← evita imagem velha em cache
```bash
docker rm -f fcg-registry 2>/dev/null; docker run -d -p 5001:5000 --name fcg-registry registry:2
curl -s http://localhost:5001/v2/_catalog >/dev/null && echo "registry OK" || echo "registry FALHOU (docker no ar?)"
export TAG=build-$(date +%s)
for i in users catalog payments; do
  docker tag  fcg-$i-api:latest localhost:5001/fcg-$i-api:$TAG
  docker push localhost:5001/fcg-$i-api:$TAG
done
echo "TAG=$TAG"
```
✅ 3 pushes OK. **Anote o TAG.**

## 4) Aplicar a plataforma no k8s — SEM o 07 (T1)
```bash
cd fcg-orchestration
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/03-secrets.yaml
kubectl apply -f k8s/01-postgres.yaml -f k8s/02-rabbitmq.yaml -f k8s/08-redis.yaml -f k8s/09-mongo.yaml
kubectl apply -f k8s/04-users-api.yaml -f k8s/05-catalog-api.yaml -f k8s/06-payments-api.yaml
kubectl apply -f k8s/08-prometheus.yaml -f k8s/09-grafana.yaml
cd ..
```
> ⚠️ **NUNCA** aplique `k8s/07-notifications-api.yaml` (container 24/7 antigo).

## 5) Apontar as APIs para a imagem fresca + rollout (T1)
```bash
for i in users catalog payments; do
  kubectl set image -n fcg deploy/$i-api $i-api=localhost:5001/fcg-$i-api:$TAG
done
kubectl rollout status -n fcg deploy/users-api
kubectl rollout status -n fcg deploy/catalog-api
kubectl rollout status -n fcg deploy/payments-api
```
> kind/minikube: pule o passo 3 e 5; use `kind load docker-image fcg-$i-api:latest` (ou `minikube image load`) e mantenha os manifestos como estão.

## 6) Esperar tudo pronto (T1)
```bash
kubectl get pods -n fcg
```
✅ Todos `Running` e `1/1` (mongo e rabbitmq sem CrashLoop).

## 7) Conferir a ponte de filas (automática) (T1)
```bash
kubectl port-forward -n fcg svc/rabbitmq-service 15672:15672 &
curl -s -u guest:guest http://localhost:15672/api/queues/%2F | grep -o 'notifications-[a-z-]*'
```
✅ `notifications-user-created` e `notifications-payment-processed`.

## 7.1) Subir o API Gateway (Kong) (T1)
```bash
cd fcg-orchestration
kubectl apply -f k8s/10-kong-postgres.yaml -f k8s/11-kong-migrations.yaml \
              -f k8s/12-kong.yaml          -f k8s/13-kong-config.yaml
kubectl wait -n fcg --for=condition=complete job/kong-migrations --timeout=180s
kubectl rollout status -n fcg deploy/kong --timeout=180s
kubectl wait -n fcg --for=condition=complete job/kong-config --timeout=180s
cd ..
# validar rotas
kubectl port-forward -n fcg svc/kong-admin-service 8001:8001 &
curl -s http://localhost:8001/routes | python3 -c "import sys,json;print([r['name'] for r in json.load(sys.stdin)['data']])"
```
✅ Jobs `kong-migrations`/`kong-config` = **Completed**, pod `kong` **1/1**; rotas `auth-route`/`users-route`/`catalog-*` listadas.
> GUI opcional (Kong Manager): encaminhe as DUAS portas — `kubectl port-forward -n fcg svc/kong-admin-service 8001:8001 &` e `kubectl port-forward -n fcg deploy/kong 8002:8002 &` → http://localhost:8002 (a GUI chama o Admin API na 8001). Ou veja o Admin API direto: http://localhost:8001/services.

## 8) Storage azurite (T2)
```bash
docker run -d --name fcg-azurite -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite
```

## 9) Túneis p/ a Function (T2 — deixe rodando)
```bash
kubectl port-forward -n fcg svc/rabbitmq-service 5672:5672 &
kubectl port-forward -n fcg svc/mongo-service    27017:27017 &
```

## 10) Subir a Function (T3 — deixe VISÍVEL na demo)
```bash
cd fcg-notifications-function/src/FCG.Notifications.Functions
cp local.settings.json.example local.settings.json
func start
```
✅ Lista `OnUserCreated`/`OnPaymentProcessed` (rabbitMQTrigger) + `Health`.

## 11) Túneis de acesso (T4)
```bash
kubectl port-forward -n fcg svc/kong-proxy-service  8000:80 &   # ENTRADA ÚNICA (gateway)
kubectl port-forward -n fcg svc/users-api-service   8081:80 &   # acesso direto (fallback)
kubectl port-forward -n fcg svc/catalog-api-service 8082:80 &
kubectl port-forward -n fcg svc/grafana    3000:3000 &
kubectl port-forward -n fcg svc/prometheus 9090:9090 &
```

## 12) Exercitar (T5) — **pelo gateway** (`:8000`) ou direto nas APIs (`:8081/:8082`)
> Para o item 1 do PDF, use o **gateway** (`http://localhost:8000`). No gateway, o cadastro exige token:
> logue como admin (`admin@fcg.com` / `TrocarEm@2026`) e cadastre o usuário comum **com o token do admin**.
> Detalhe passo a passo do fluxo pelo gateway (GW.1–GW.5): ver `ROTEIRO_TESTE_E2E_HIBRIDO.md` (Parte C-GW).
- **Cadastro** → boas-vindas no T3:
```bash
curl -s -X POST http://localhost:8081/api/usuarios/registrar -H "content-type: application/json" \
  -d '{"nome":"Raul Demo","email":"raul.demo@fcg.com","senha":"Senha@123"}'
```
- **Admin + jogo** (copie o accessToken e o id do jogo):
```bash
curl -s -X POST http://localhost:8081/api/usuarios/administradores -H "content-type: application/json" \
  -d '{"nome":"Admin","email":"admin@fcg.com","senha":"Senha@123"}'
curl -s -X POST http://localhost:8081/api/auth/login -H "content-type: application/json" \
  -d '{"email":"admin@fcg.com","senha":"Senha@123"}'          # -> ADMIN token
curl -s -X POST http://localhost:8082/api/jogos -H "authorization: Bearer <ADMIN>" -H "content-type: application/json" \
  -d '{"titulo":"Elden Ring","descricao":"RPG","genero":"RPG","preco":199.90}'   # -> id do jogo
```
- **Login user + compra** → confirmação no T3:
```bash
curl -s -X POST http://localhost:8081/api/auth/login -H "content-type: application/json" \
  -d '{"email":"raul.demo@fcg.com","senha":"Senha@123"}'      # -> USER token
curl -s -X POST http://localhost:8082/api/biblioteca/aquisicoes -H "authorization: Bearer <USER>" -H "content-type: application/json" \
  -d '{"jogoId":"<ID_DO_JOGO>"}'
```
✅ Cadastro **201** + log de boas-vindas; compra **202** + log de confirmação (Preço R$ 199.90).

## 13) Visualizar
```bash
# NoSQL/idempotência
kubectl exec -n fcg deploy/mongo -- mongosh --quiet fcg_notifications \
  --eval 'db.processed_events.find({},{_id:0,consumer:1,messageId:1}).toArray()'
# Cache Redis (consulte o catálogo antes p/ gerar as chaves)
curl -s http://localhost:8082/api/jogos >/dev/null
kubectl exec -n fcg deploy/redis -- redis-cli KEYS 'catalog:*'
```
- **Grafana:** http://localhost:3000 (`admin`/`admin1234`) → Dashboards → **"FCG - Observabilidade das APIs"** (ou `/d/fcg-apis`). **Não** use Drilldown→Traces.
- **Prometheus:** http://localhost:9090 → Status → Target health → `users-api`/`catalog-api` **UP**.

## 14) Encerrar
```bash
# Ctrl+C no func (T3) e nos port-forwards
docker rm -f fcg-azurite
kubectl delete namespace fcg
# (o fcg-registry pode ficar; na próxima subida o passo 3 reusa/recria)
```

---

## Notas Windows (PowerShell) — se NÃO usar Git Bash/WSL
Traduza só os 2 loops; o resto é igual.
```powershell
# passo 3
$TAG = "build-$([int](Get-Date -UFormat %s))"
foreach ($i in "users","catalog","payments") {
  docker tag  "fcg-$i-api:latest" "localhost:5001/fcg-$i-api:$TAG"
  docker push "localhost:5001/fcg-$i-api:$TAG"
}
# passo 5
foreach ($i in "users","catalog","payments") {
  kubectl set image -n fcg deploy/$i-api $i-api="localhost:5001/fcg-$i-api:$TAG"
}
```
Port-forwards: no PowerShell, abra **um terminal por port-forward** (não use `&`). `chmod` do func: **não se aplica** no Windows.
</content>
