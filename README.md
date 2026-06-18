# skipa-infra

SKIPA 서비스의 Kubernetes manifest와 Argo CD Application 설정을 관리하는 레포지토리입니다.

CI는 각 서비스 레포의 GitHub Actions가 담당하고, CD는 이 레포를 기준으로 Argo CD가 담당합니다. 서비스 레포의 GitHub Actions는 Docker image를 Harbor에 push한 뒤 이 레포의 `k8s/{service}/kustomization.yml`에 image tag를 반영합니다.

## 디렉토리 구조

```text
skipa-infra/
├── argocd/
│   ├── root/
│   │   └── skipa-root-application.yml
│   └── apps/
│       ├── datastore-application.yml
│       ├── queue-application.yml
│       ├── backend-application.yml
│       ├── ai-application.yml
│       ├── frontend-application.yml
│       └── monitoring-application.yml
└── k8s/
    ├── datastore/
    │   ├── kustomization.yml
    │   ├── postgres-service.yml
    │   ├── postgres-statefulset.yml
    │   ├── redis-service.yml
    │   ├── redis-statefulset.yml
    │   ├── minio-service.yml
    │   ├── minio-statefulset.yml
    │   ├── minio-ingress.yml
    │   ├── qdrant-service.yml
    │   └── qdrant-statefulset.yml
    ├── queue/
    │   ├── kustomization.yml
    │   ├── rabbitmq-service.yml
    │   └── rabbitmq-statefulset.yml
    ├── backend/
    │   ├── kustomization.yml
    │   ├── configmap.yml
    │   ├── deployment.yml
    │   ├── service.yml
    │   └── ingress.yml
    ├── ai/
    │   ├── kustomization.yml
    │   ├── configmap.yml
    │   ├── deployment.yml
    │   ├── report-worker-deployment.yml
    │   ├── patent-extract-worker-deployment.yml
    │   ├── pre-evaluation-worker-deployment.yml
    │   ├── service.yml
    │   └── ingress.yml
    ├── frontend/
    │   ├── kustomization.yml
    │   ├── deployment.yml
    │   ├── service.yml
    │   └── ingress.yml
    └── monitoring/
        ├── kustomization.yml
        ├── prometheus-rbac.yml
        ├── prometheus-configmap.yml
        ├── prometheus-pvc.yml
        ├── prometheus-deployment.yml
        ├── prometheus-service.yml
        ├── grafana-datasource-configmap.yml
        ├── grafana-dashboard-provider-configmap.yml
        ├── grafana-dashboard-configmap.yml
        ├── grafana-pvc.yml
        ├── grafana-deployment.yml
        ├── grafana-service.yml
        └── grafana-ingress.yml
```

## Namespace

서비스 리소스는 아래 namespace에 배포합니다.

```text
skala3-finalproj-class2-team8
```

Argo CD Application은 공용 Argo CD namespace에 생성합니다.

```text
skala-argocd
```

각 child Application은 `CreateNamespace=false`를 사용하므로 서비스 namespace는 미리 존재해야 합니다.

## 배포 구조

SKIPA는 Argo CD App of Apps 구조를 사용합니다.

```text
team8-skipa (root Application)
├── team8-skipa-datastore  # PostgreSQL / Redis / MinIO / Qdrant
├── team8-skipa-queue      # RabbitMQ
├── team8-skipa-backend    # Spring Boot backend
├── team8-skipa-ai         # FastAPI AI API + workers
├── team8-skipa-frontend   # frontend
└── team8-skipa-monitoring # Prometheus / Grafana
```

root Application은 `argocd/apps/` 디렉토리를 바라보고, 하위 Application들을 생성 및 관리합니다. 각 하위 Application은 자기 서비스의 `k8s/{service}` 디렉토리를 바라봅니다.

```text
team8-skipa-datastore  -> k8s/datastore
team8-skipa-queue      -> k8s/queue
team8-skipa-backend    -> k8s/backend
team8-skipa-ai         -> k8s/ai
team8-skipa-frontend   -> k8s/frontend
team8-skipa-monitoring -> k8s/monitoring
```

## Sync 순서

하위 Application에는 sync wave를 지정합니다.

```text
wave 0: datastore, queue
wave 1: backend, ai
wave 2: frontend
wave 3: monitoring
```

Datastore와 Queue가 먼저 생성되고, 이후 backend/AI 서버, frontend, monitoring이 동기화되는 흐름입니다.

Datastore와 Queue는 상태 저장 리소스를 포함하므로 `prune: false`, `selfHeal: true`로 관리합니다. Backend, AI, Frontend, Monitoring은 `prune: true`, `selfHeal: true`로 관리합니다.

## 외부 엔드포인트

Ingress는 `public-nginx` class와 `letsencrypt-prod` ClusterIssuer를 사용합니다.

```text
Frontend: https://team8-skipa.skala25a.project.skala-ai.com
Backend:  https://api-team8-skipa.skala25a.project.skala-ai.com
AI:       https://ai-team8-skipa.skala25a.project.skala-ai.com
MinIO:    https://minio-team8-skipa.skala25a.project.skala-ai.com
Grafana:  https://grafana-team8-skipa.skala25a.project.skala-ai.com
```

MinIO ingress는 API port `9000`으로 연결됩니다. Console port `9001`은 ingress로 공개하지 않습니다.

## 리소스 구성

### Frontend

- Deployment: `skipa-frontend`
- Service: `skipa-frontend`
- Ingress: `skipa-frontend-ingress`
- Container Port: `80`
- Service Port: `80`
- Image: `amdp-registry.skala-ai.com/skala26a-ai2/skipa-frontend:<tag>`

### Backend

- ConfigMap: `skipa-backend-config`
- Deployment: `skipa-backend`
- Service: `skipa-backend`
- Ingress: `skipa-backend-ingress`
- Port: `8080`
- Image: `amdp-registry.skala-ai.com/skala26a-ai2/skipa-backend:<tag>`

Backend는 PostgreSQL, Redis, RabbitMQ, MinIO, AI API를 사용합니다.

```text
PostgreSQL: skipa-postgres:5432
Redis:      skipa-redis:6379
RabbitMQ:   skipa-rabbitmq:5672
MinIO:      http://skipa-minio:9000
AI API:     http://skipa-ai:8000
```

### AI

- ConfigMap: `skipa-ai-config`
- API Deployment: `skipa-ai`
- Worker Deployments:
  - `skipa-ai-report-worker`
  - `skipa-ai-patent-extract-worker`
  - `skipa-ai-pre-evaluation-worker`
- Service: `skipa-ai`
- Ingress: `skipa-ai-ingress`
- API Port: `8000`
- Image: `amdp-registry.skala-ai.com/skala26a-ai2/skipa-ai:<tag>`

AI API와 worker는 같은 이미지를 사용합니다. Worker는 container args와 `APP_SERVICE` 값으로 실행 모드를 구분합니다.

```text
report-worker          -> skipa.report.generate
patent-extract-worker  -> skipa.patent-extract
pre-evaluation-worker  -> skipa.pre-evaluation.generate
```

AI는 MinIO, Qdrant, RabbitMQ, Backend internal API를 사용합니다.

```text
MinIO:       http://skipa-minio:9000
Qdrant HTTP: http://skipa-qdrant:6333
Qdrant gRPC: skipa-qdrant:6334
RabbitMQ:    skipa-rabbitmq:5672
Backend API: http://skipa-backend:8080/api/v1
```

### PostgreSQL

- Service: `skipa-postgres`
- Headless Service: `skipa-postgres-headless`
- StatefulSet: `skipa-postgres`
- Secret: `skipa-postgres-secret`
- Image: `postgres:16`
- Port: `5432`
- StorageClass: `gp3`
- Storage: `2Gi`

### Redis

- Service: `skipa-redis`
- Headless Service: `skipa-redis-headless`
- StatefulSet: `skipa-redis`
- Secret: `skipa-redis-secret`
- Image: `redis:7-alpine`
- Port: `6379`
- StorageClass: `gp3`
- Storage: `1Gi`

### MinIO

- Service: `skipa-minio`
- Headless Service: `skipa-minio-headless`
- StatefulSet: `skipa-minio`
- Ingress: `skipa-minio-ingress`
- Secret: `skipa-minio-secret`
- Image: `minio/minio:latest`
- API Port: `9000`
- Console Port: `9001`
- StorageClass: `gp3`
- Storage: `2Gi`

### Qdrant

- Service: `skipa-qdrant`
- Headless Service: `skipa-qdrant-headless`
- StatefulSet: `skipa-qdrant`
- Secret: `skipa-qdrant-secret`
- Image: `qdrant/qdrant:latest`
- HTTP Port: `6333`
- gRPC Port: `6334`
- StorageClass: `gp3`
- Storage: `2Gi`

### RabbitMQ

- Service: `skipa-rabbitmq`
- Headless Service: `skipa-rabbitmq-headless`
- StatefulSet: `skipa-rabbitmq`
- Secret: `skipa-rabbitmq-secret`
- Image: `rabbitmq:3.13-management-alpine`
- AMQP Port: `5672`
- Management Port: `15672`
- StorageClass: `gp3`
- Storage: `1Gi`

### Monitoring

- Prometheus:
  - ServiceAccount: `skipa-prometheus`
  - ClusterRole / ClusterRoleBinding: `skipa-prometheus`
  - ConfigMap: `skipa-prometheus-config`
  - Deployment: `skipa-prometheus`
  - Service: `skipa-prometheus`
  - PVC: `skipa-prometheus-data`
  - Image: `prom/prometheus:v2.55.1`
  - Port: `9090`
  - StorageClass: `gp3`
  - Storage: `5Gi`
- Grafana:
  - ConfigMap: `skipa-grafana-datasources`
  - ConfigMap: `skipa-grafana-dashboard-providers`
  - ConfigMap: `skipa-grafana-dashboards`
  - Deployment: `skipa-grafana`
  - Service: `skipa-grafana`
  - Ingress: `skipa-grafana-ingress`
  - Secret: `skipa-grafana-secret`
  - PVC: `skipa-grafana-data`
  - Image: `grafana/grafana:11.3.0`
  - Port: `3000`
  - StorageClass: `gp3`
  - Storage: `2Gi`

Prometheus는 Kubernetes API, node/cAdvisor metric, annotation이 붙은 Pod/Service metric을 수집합니다. Backend Service는 `/actuator/prometheus`, AI Service는 `/metrics`를 scrape 대상으로 등록합니다.

Grafana는 Prometheus datasource와 `SKIPA Monitoring Overview` 기본 dashboard를 provisioning합니다.

## Kubernetes 리소스 역할

### Service

Kubernetes에서 Pod는 재시작되면 IP가 바뀔 수 있습니다. Service는 Pod 앞에 고정된 내부 접속 주소를 제공합니다.

```text
PostgreSQL: skipa-postgres:5432
Redis: skipa-redis:6379
MinIO: skipa-minio:9000
Qdrant: skipa-qdrant:6333 / 6334
RabbitMQ: skipa-rabbitmq:5672
Backend: skipa-backend:8080
AI: skipa-ai:8000
Frontend: skipa-frontend:80
Prometheus: skipa-prometheus:9090
Grafana: skipa-grafana:3000
```

### Headless Service

StatefulSet이 각 Pod를 안정적으로 식별하기 위해 사용하는 Service입니다. 일반적인 애플리케이션 접속용 주소로는 headless Service가 아니라 일반 Service를 사용합니다.

### StatefulSet

PostgreSQL, Redis, MinIO, Qdrant, RabbitMQ처럼 상태를 가진 애플리케이션은 Pod가 재시작되어도 같은 저장소를 다시 사용해야 합니다. StatefulSet은 고정된 Pod 이름과 PVC를 기반으로 상태가 있는 애플리케이션을 안정적으로 실행합니다.

```text
skipa-postgres-0
skipa-redis-0
skipa-minio-0
skipa-qdrant-0
skipa-rabbitmq-0
```

### Deployment

Backend, Frontend, AI API, AI worker처럼 상태를 직접 저장하지 않는 애플리케이션은 Deployment로 실행합니다. 새 이미지 배포 시 RollingUpdate를 통해 기존 Pod를 유지하면서 새 Pod로 교체합니다.

Prometheus와 Grafana도 Deployment로 실행하지만, 수집 데이터와 Grafana 설정을 유지하기 위해 PVC를 사용합니다.

## Secret 생성

실제 비밀번호, 토큰, 인증 정보는 GitHub에 커밋하지 않습니다. 아래 Secret들은 클러스터에 직접 생성해야 합니다.

명령 예시는 서비스 namespace 기준입니다.

```bash
NAMESPACE=skala3-finalproj-class2-team8
```

### PostgreSQL Secret

```bash
POSTGRES_PASSWORD=$(openssl rand -hex 24)

kubectl create secret generic skipa-postgres-secret \
  -n "$NAMESPACE" \
  --from-literal=username="skipa" \
  --from-literal=password="$POSTGRES_PASSWORD" \
  --from-literal=database="skipa"
```

### Redis Secret

```bash
REDIS_PASSWORD=$(openssl rand -hex 24)

kubectl create secret generic skipa-redis-secret \
  -n "$NAMESPACE" \
  --from-literal=password="$REDIS_PASSWORD"
```

### MinIO Secret

```bash
MINIO_ROOT_PASSWORD=$(openssl rand -hex 24)

kubectl create secret generic skipa-minio-secret \
  -n "$NAMESPACE" \
  --from-literal=root-user="skipa" \
  --from-literal=root-password="$MINIO_ROOT_PASSWORD"
```

### Qdrant Secret

```bash
QDRANT_API_KEY=$(openssl rand -hex 32)

kubectl create secret generic skipa-qdrant-secret \
  -n "$NAMESPACE" \
  --from-literal=api-key="$QDRANT_API_KEY"
```

### RabbitMQ Secret

```bash
RABBITMQ_PASSWORD=$(openssl rand -hex 24)

kubectl create secret generic skipa-rabbitmq-secret \
  -n "$NAMESPACE" \
  --from-literal=username="skipa" \
  --from-literal=password="$RABBITMQ_PASSWORD"
```

### Backend Secret

Backend와 AI가 공유하는 internal API key는 같은 값이어야 합니다.

```bash
JWT_SECRET=$(openssl rand -base64 64)
INTERNAL_API_KEY=$(openssl rand -hex 32)

kubectl create secret generic skipa-backend-secret \
  -n "$NAMESPACE" \
  --from-literal=JWT_SECRET="$JWT_SECRET" \
  --from-literal=INTERNAL_API_KEY="$INTERNAL_API_KEY"
```

### AI Secret

`skipa-ai-secret`은 AI 서비스가 사용하는 외부 API key 등을 담습니다. 필요한 key 목록은 AI 애플리케이션 설정과 맞춰야 합니다.

예시:

```bash
kubectl create secret generic skipa-ai-secret \
  -n "$NAMESPACE" \
  --from-literal=OPENAI_API_KEY="<OPENAI_API_KEY>" \
  --from-literal=TAVILY_API_KEY="<TAVILY_API_KEY>"
```

### Grafana Secret

Grafana 관리자 계정 정보입니다.

```bash
GRAFANA_ADMIN_PASSWORD=$(openssl rand -hex 16)

kubectl create secret generic skipa-grafana-secret \
  -n "$NAMESPACE" \
  --from-literal=admin-user="admin" \
  --from-literal=admin-password="$GRAFANA_ADMIN_PASSWORD"
```

### Harbor Image Pull Secret

Harbor private registry에서 이미지를 pull하기 위한 Secret입니다.

```bash
kubectl create secret docker-registry harbor-registry-secret \
  -n "$NAMESPACE" \
  --docker-server=amdp-registry.skala-ai.com \
  --docker-username=<HARBOR_USERNAME> \
  --docker-password='<HARBOR_PASSWORD>' \
  --docker-email=<YOUR_EMAIL>
```

필요한 Secret 목록:

```text
harbor-registry-secret
skipa-backend-secret
skipa-ai-secret
skipa-postgres-secret
skipa-redis-secret
skipa-minio-secret
skipa-qdrant-secret
skipa-rabbitmq-secret
skipa-grafana-secret
```

Secret 생성 확인:

```bash
kubectl get secret -n "$NAMESPACE"
```

## Argo CD 적용

root Application 하나만 수동으로 적용합니다. 이후 `argocd/apps/` 디렉토리의 child Application들은 Argo CD가 자동으로 생성 및 관리합니다.

```bash
kubectl apply -f argocd/root/skipa-root-application.yml
```

확인:

```bash
kubectl get application -n skala-argocd
kubectl describe application team8-skipa -n skala-argocd
```

## Kustomize 사용 방식

각 서비스 디렉토리는 `kustomization.yml`을 포함합니다. Argo CD child Application은 각 `k8s/{service}` 디렉토리를 바라보고, 해당 디렉토리의 Kustomize 설정을 기준으로 리소스를 동기화합니다.

예시:

```text
k8s/backend/
├── kustomization.yml
├── configmap.yml
├── deployment.yml
├── service.yml
└── ingress.yml
```

`deployment.yml`에는 기본 image가 들어 있고, 실제 배포 시에는 `kustomization.yml`의 `images.newTag`를 GitHub Actions가 변경합니다.

```yaml
images:
  - name: amdp-registry.skala-ai.com/skala26a-ai2/skipa-backend
    newName: amdp-registry.skala-ai.com/skala26a-ai2/skipa-backend
    newTag: dev-a1b2c3d
```

## 이미지 태그 전략

`dev-latest`만 사용하면 Git manifest가 바뀌지 않아 Argo CD가 변경을 감지하지 못할 수 있습니다. 따라서 실제 Argo CD 배포 기준은 `kustomization.yml`에 기록된 고유 태그입니다.

```text
skipa-backend:dev-<short-sha>
skipa-frontend:dev-<short-sha>
skipa-ai:dev-<short-sha>
```

서비스 레포의 GitHub Actions는 다음 두 태그를 함께 push할 수 있습니다.

```text
dev-<short-sha>  # 실제 배포에 사용
dev-latest       # 참고용 또는 수동 확인용
```

## 배포 흐름

각 서비스 레포에서 main 브랜치에 merge되면 GitHub Actions가 실행됩니다. GitHub Actions는 이미지를 빌드하고 Harbor에 push한 뒤, 이 레포의 해당 서비스 `kustomization.yml` image tag를 업데이트합니다. Argo CD는 이 레포의 변경을 감지하여 클러스터에 자동으로 sync합니다.

Backend 예시:

```text
skipa-backend main merge
-> GitHub Actions 실행 (skipa-backend 레포)
-> Spring Boot jar 빌드
-> Docker image 빌드
-> Harbor push
-> skipa-infra/k8s/backend/kustomization.yml newTag 수정 후 commit
-> Argo CD가 skipa-infra 변경 감지
-> team8-skipa-backend Application sync
-> RollingUpdate로 backend Pod 교체
```

Frontend, AI도 동일한 방식으로 각각 자기 디렉토리의 `kustomization.yml`만 수정합니다.

```text
skipa-frontend -> k8s/frontend/kustomization.yml
skipa-backend  -> k8s/backend/kustomization.yml
skipa-ai       -> k8s/ai/kustomization.yml
```

Datastore와 Queue는 서비스 배포 자동화 대상에서 제외합니다.

## GitHub Secrets

각 서비스 레포의 GitHub Actions에서 Harbor에 이미지를 push하고 이 레포에 commit하기 위해 아래 secret이 필요합니다.

```text
HARBOR_USERNAME
HARBOR_PASSWORD
INFRA_REPO_TOKEN
```

등록 위치:

```text
GitHub Repository
-> Settings
-> Secrets and variables
-> Actions
-> New repository secret
```

## 환경변수

Backend와 AI는 ConfigMap과 Secret을 함께 사용합니다.

### Backend ConfigMap

주요 설정:

```text
SPRING_PROFILES_ACTIVE=prod
DB_HOST=skipa-postgres
DB_PORT=5432
REDIS_HOST=skipa-redis
REDIS_PORT=6379
RABBITMQ_HOST=skipa-rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_EXCHANGE=skipa.exchange
MINIO_ENDPOINT=http://skipa-minio:9000
MINIO_PUBLIC_ENDPOINT=https://minio-team8-skipa.skala25a.project.skala-ai.com
MINIO_BUCKET=skipa
AI_SERVER_BASE_URL=http://skipa-ai:8000
```

Backend Secret에서 주입되는 값:

```text
JWT_SECRET
INTERNAL_API_KEY
DB_NAME
DB_USERNAME
DB_PASSWORD
REDIS_PASSWORD
RABBITMQ_USERNAME
RABBITMQ_PASSWORD
MINIO_ACCESS_KEY
MINIO_SECRET_KEY
```

### AI ConfigMap

주요 설정:

```text
APP_SERVICE=api
PORT=8000
PUBLIC_FILE_BASE_URL=https://ai-team8-skipa.skala25a.project.skala-ai.com/files
INTENT_PROVIDER=openai
ANSWER_PROVIDER=openai
EMBEDDING_PROVIDER=openai
OPENAI_INTENT_MODEL=gpt-4.1-mini
OPENAI_ANSWER_MODEL=gpt-4.1
OPENAI_EMBEDDING_MODEL=text-embedding-3-large
VECTOR_STORE=qdrant
QDRANT_URL=http://skipa-qdrant:6333
RABBITMQ_HOST=skipa-rabbitmq
RABBITMQ_PORT=5672
BACKEND_INTERNAL_BASE_URL=http://skipa-backend:8080/api/v1
```

AI Secret과 다른 Secret에서 주입되는 값:

```text
skipa-ai-secret 전체
MINIO_ACCESS_KEY
MINIO_SECRET_KEY
QDRANT_API_KEY
RABBITMQ_USERNAME
RABBITMQ_PASSWORD
INTERNAL_API_KEY
```

## 상태 확인

서비스 namespace의 전체 상태를 확인합니다.

```bash
kubectl get pods,svc,ingress,pvc -n skala3-finalproj-class2-team8
```

정상 Pod 예시:

```text
skipa-postgres-0                    1/1   Running
skipa-redis-0                       1/1   Running
skipa-minio-0                       1/1   Running
skipa-qdrant-0                      1/1   Running
skipa-rabbitmq-0                    1/1   Running
skipa-backend-...                   1/1   Running
skipa-ai-...                        1/1   Running
skipa-ai-report-worker-...          1/1   Running
skipa-ai-patent-extract-worker-...  1/1   Running
skipa-ai-pre-evaluation-worker-...  1/1   Running
skipa-frontend-...                  1/1   Running
skipa-prometheus-...                1/1   Running
skipa-grafana-...                   1/1   Running
```

PVC 예시:

```text
postgres-data-skipa-postgres-0   Bound   2Gi
redis-data-skipa-redis-0         Bound   1Gi
minio-data-skipa-minio-0         Bound   2Gi
qdrant-data-skipa-qdrant-0       Bound   2Gi
rabbitmq-data-skipa-rabbitmq-0   Bound   1Gi
skipa-prometheus-data            Bound   5Gi
skipa-grafana-data               Bound   2Gi
```

## 로컬 확인

PostgreSQL, Redis, Qdrant, RabbitMQ Management UI, MinIO Console처럼 외부에 직접 공개하지 않는 포트는 `port-forward`로 확인합니다.

### Backend

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-backend 18080:8080
```

```text
http://127.0.0.1:18080
```

### AI

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-ai 18000:8000
```

```text
http://127.0.0.1:18000
```

### PostgreSQL

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-postgres 15432:5432
```

DataGrip 연결 정보:

```text
Host: 127.0.0.1
Port: 15432
Database: skipa
User: skipa
Password: skipa-postgres-secret의 password 값
```

### Redis

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-redis 16379:6379
redis-cli -h 127.0.0.1 -p 16379 -a '<REDIS_PASSWORD>' PING
```

### Qdrant

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-qdrant 16333:6333
```

```text
http://127.0.0.1:16333
```

### RabbitMQ Management UI

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-rabbitmq 15672:15672
```

```text
http://127.0.0.1:15672
```

### MinIO Console

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-minio 19001:9001
```

```text
http://127.0.0.1:19001
```

### Prometheus

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-prometheus 19090:9090
```

```text
http://127.0.0.1:19090
```

### Grafana

Grafana는 ingress로 접근할 수 있습니다.

```text
https://grafana-team8-skipa.skala25a.project.skala-ai.com
```

로컬에서 확인할 때는 `port-forward`를 사용할 수 있습니다.

```bash
kubectl port-forward -n skala3-finalproj-class2-team8 svc/skipa-grafana 13000:3000
```

```text
http://127.0.0.1:13000
```

## Flyway 마이그레이션

DB schema migration은 backend 애플리케이션의 Flyway가 담당합니다.

```text
team8-skipa-datastore
-> PostgreSQL / Redis / MinIO / Qdrant 실행

team8-skipa-backend
-> Spring Boot 실행
-> DB 연결
-> Flyway migration 실행
-> 애플리케이션 실행
```

따라서 Datastore를 Argo CD로 관리하더라도 Flyway 구조는 그대로 유지할 수 있습니다.

## 주의사항

- Secret 값은 GitHub에 커밋하지 않습니다.
- `.env` 파일은 GitHub에 커밋하지 않습니다.
- Harbor 비밀번호, JWT Secret, external API key는 코드에 직접 작성하지 않습니다.
- PostgreSQL, Redis, Qdrant, RabbitMQ, MinIO Console은 외부에 직접 공개하지 않고 필요할 때 `port-forward`로 확인합니다.
- MinIO API는 `minio-team8-skipa.skala25a.project.skala-ai.com` ingress로 공개됩니다.
- PVC를 삭제하면 연결된 실제 볼륨 데이터도 삭제될 수 있으므로 주의합니다.
- `gp3` StorageClass의 ReclaimPolicy가 `Delete`이면 PVC 삭제 시 데이터가 함께 삭제될 수 있습니다.
- Datastore와 Queue는 Argo CD로 관리하지만, 서비스 배포 자동화 대상에서는 제외합니다.
- Frontend, Backend, AI 배포 시에는 각 서비스의 `kustomization.yml` image tag만 변경합니다.
