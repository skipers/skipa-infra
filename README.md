# skipa-infra

이 레포지토리는 SKIPA 서비스의 Kubernetes manifest 및 Argo CD 설정을 관리합니다.

CI는 각 서비스 레포의 GitHub Actions가 담당하고, CD는 이 레포를 기준으로 Argo CD가 담당합니다.
서비스 배포 시 각 서비스 레포의 GitHub Actions는 Docker image를 빌드해 Harbor에 push한 뒤, 이 레포의 해당 서비스 `kustomization.yml`에 image tag를 반영합니다.

## 디렉토리 구조

```text
skipa-infra/
├── argocd/
│   ├── root/
│   │   └── skipa-root-application.yml
│   └── apps/
│       ├── datastore-application.yml
│       ├── backend-application.yml
│       ├── frontend-application.yml
│       └── ai-application.yml
└── k8s/
    ├── datastore/
    │   ├── kustomization.yml
    │   ├── postgres-service.yml
    │   ├── postgres-statefulset.yml
    │   ├── redis-service.yml
    │   └── redis-statefulset.yml
    ├── backend/
    │   ├── kustomization.yml
    │   ├── deployment.yml
    │   └── service.yml
    ├── frontend/
    │   ├── kustomization.yml
    │   ├── deployment.yml
    │   └── service.yml
    └── ai/
        ├── kustomization.yml
        ├── deployment.yml
        └── service.yml
```

## Namespace

SKIPA 8팀은 아래 namespace를 사용합니다.

```text
skala3-finalproj-class2-team8
```

Argo CD Application은 공용 Argo CD namespace에 생성합니다.

```text
skala-argocd
```

## 배포 구조

SKIPA는 Argo CD App of Apps 구조를 사용합니다.

```text
team8-skipa (root Application)
├── team8-skipa-datastore  # PostgreSQL / Redis
├── team8-skipa-backend    # Spring Boot backend
├── team8-skipa-ai         # FastAPI AI server
└── team8-skipa-frontend   # frontend
```

root Application은 `argocd/apps/` 디렉토리를 바라보고, 하위 Application들을 생성 및 관리합니다.
각 하위 Application은 자기 서비스의 `k8s/{service}` 디렉토리를 바라봅니다.

```text
team8-skipa-datastore  → k8s/datastore
team8-skipa-backend    → k8s/backend
team8-skipa-ai         → k8s/ai
team8-skipa-frontend   → k8s/frontend
```

## Sync 순서

하위 Application에는 sync wave를 지정합니다.

```text
wave 0: datastore
wave 1: backend, ai
wave 2: frontend
```

Datastore가 먼저 생성되고, 이후 backend/AI 서버와 frontend가 동기화되는 흐름입니다.

## 리소스 구성

### Backend

- Deployment: `skipa-backend`
- Service: `skipa-backend`
- Port: `8080`
- Image Registry: Harbor
- Image: `amdp-registry.skala-ai.com/skala3-finalproj-class2-team8/skipa-backend:<tag>`

Service는 `ClusterIP` 타입으로 생성합니다.
외부에 직접 공개하지 않고, 클러스터 내부에서 접근하거나 로컬 확인 시 `port-forward`를 사용합니다.

```bash
kubectl port-forward svc/skipa-backend 18080:8080
```

로컬 확인 주소:

```text
http://127.0.0.1:18080
```

### Frontend

- Deployment: `skipa-frontend`
- Service: `skipa-frontend`
- Port: `3000`
- Image Registry: Harbor
- Image: `amdp-registry.skala-ai.com/skala3-finalproj-class2-team8/skipa-frontend:<tag>`

### AI

- Deployment: `skipa-ai`
- Service: `skipa-ai`
- Port: `8000`
- Image Registry: Harbor
- Image: `amdp-registry.skala-ai.com/skala3-finalproj-class2-team8/skipa-ai:<tag>`

### PostgreSQL

- Service: `skipa-postgres`
- Headless Service: `skipa-postgres-headless`
- StatefulSet: `skipa-postgres`
- Secret: `skipa-postgres-secret`
- Port: `5432`
- StorageClass: `gp3`
- Storage: `2Gi`

백엔드 내부 접속 주소:

```text
skipa-postgres:5432
```

### Redis

- Service: `skipa-redis`
- Headless Service: `skipa-redis-headless`
- StatefulSet: `skipa-redis`
- Secret: `skipa-redis-secret`
- Port: `6379`
- StorageClass: `gp3`
- Storage: `1Gi`

백엔드 내부 접속 주소:

```text
skipa-redis:6379
```

## Kubernetes 리소스 역할

### Service

Kubernetes에서 Pod는 재시작되면 IP가 바뀔 수 있습니다.
Service는 Pod 앞에 고정된 내부 접속 주소를 제공합니다.

백엔드는 PostgreSQL/Redis Pod의 IP를 직접 바라보지 않고, 아래 Service 주소로 접근합니다.

```text
PostgreSQL: skipa-postgres:5432
Redis: skipa-redis:6379
```

### Headless Service

StatefulSet이 각 Pod를 안정적으로 식별하기 위해 사용하는 Service입니다.
일반적인 백엔드 접속용 주소로는 `skipa-postgres`, `skipa-redis` Service를 사용합니다.

### StatefulSet

PostgreSQL, Redis처럼 데이터를 저장하는 애플리케이션은 Pod가 재시작되어도 같은 저장소를 다시 사용해야 합니다.
StatefulSet은 고정된 Pod 이름과 PVC를 기반으로 상태가 있는 애플리케이션을 안정적으로 실행합니다.

예시:

```text
skipa-postgres-0
skipa-redis-0
```

### Deployment

백엔드, 프론트엔드, AI 서버처럼 상태를 직접 저장하지 않는 애플리케이션 서버는 Deployment로 실행합니다.
Deployment는 새 이미지 배포 시 RollingUpdate를 통해 기존 Pod를 유지하면서 새 Pod로 교체할 수 있습니다.

## Secret 생성

실제 비밀번호, 토큰, 인증 정보는 GitHub에 커밋하지 않습니다.
아래 Secret들은 클러스터에 직접 생성해야 합니다.

### PostgreSQL Secret

```bash
POSTGRES_PASSWORD=$(openssl rand -hex 24)

kubectl create secret generic skipa-postgres-secret \
  --from-literal=username="skipa" \
  --from-literal=password="$POSTGRES_PASSWORD" \
  --from-literal=database="skipa"
```

### Redis Secret

```bash
REDIS_PASSWORD=$(openssl rand -hex 24)

kubectl create secret generic skipa-redis-secret \
  --from-literal=password="$REDIS_PASSWORD"
```

### Backend JWT Secret

```bash
JWT_SECRET=$(openssl rand -base64 64)

kubectl create secret generic skipa-backend-secret \
  --from-literal=JWT_SECRET="$JWT_SECRET"
```

### Harbor Image Pull Secret

Harbor private registry에서 이미지를 pull하기 위한 Secret입니다.

```bash
kubectl create secret docker-registry harbor-registry-secret \
  --docker-server=amdp-registry.skala-ai.com \
  --docker-username=<HARBOR_USERNAME> \
  --docker-password='<HARBOR_PASSWORD>' \
  --docker-email=<YOUR_EMAIL>
```

Secret 생성 확인:

```bash
kubectl get secret
```

필요한 Secret 목록:

```text
harbor-registry-secret
skipa-backend-secret
skipa-postgres-secret
skipa-redis-secret
```

## Argo CD 적용

root Application 하나만 수동으로 적용합니다.
이후 `argocd/apps/` 디렉토리의 child Application들은 Argo CD가 자동으로 생성 및 관리합니다.

```bash
kubectl apply -f argocd/root/skipa-root-application.yml
```

확인:

```bash
kubectl get application -n skala-argocd
kubectl describe application team8-skipa -n skala-argocd
```

## Datastore 관리 방식

Datastore도 Argo CD로 관리합니다.
다만 PostgreSQL/Redis는 상태를 가진 리소스이므로, 서비스 배포 자동화 대상과 분리합니다.

즉, `team8-skipa-datastore` Application은 `k8s/datastore`를 기준으로 PostgreSQL/Redis 리소스를 관리하지만, frontend/backend/ai 레포의 GitHub Actions는 `k8s/datastore`를 수정하지 않습니다.

```text
Argo CD 관리 대상: PostgreSQL / Redis / Service / StatefulSet / PVC
GitHub Actions 자동 태그 변경 대상: 제외
```

Datastore Application은 self-heal은 활성화하지만, prune은 비활성화합니다.

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: true
```

Datastore 상태 확인:

```bash
kubectl get pods
kubectl get svc
kubectl get pvc
```

정상 실행 예시:

```text
skipa-postgres-0   1/1   Running
skipa-redis-0      1/1   Running
```

PVC 예시:

```text
postgres-data-skipa-postgres-0   Bound   2Gi
redis-data-skipa-redis-0         Bound   1Gi
```

## Flyway 마이그레이션

DB schema migration은 backend 애플리케이션의 Flyway가 담당합니다.

```text
team8-skipa-datastore
→ PostgreSQL / Redis 실행

team8-skipa-backend
→ Spring Boot 실행
→ DB 연결
→ Flyway migration 실행
→ 애플리케이션 실행
```

따라서 Datastore를 Argo CD로 관리하더라도 Flyway 구조는 그대로 유지할 수 있습니다.

Spring Boot 설정 예시:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
      password: ${REDIS_PASSWORD}

  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false

  jpa:
    hibernate:
      ddl-auto: validate
```

## Kustomize 사용 방식

각 서비스 디렉토리는 `kustomization.yml`을 포함합니다.
Argo CD child Application은 각 `k8s/{service}` 디렉토리를 바라보고, 해당 디렉토리의 Kustomize 설정을 기준으로 리소스를 동기화합니다.

예시:

```text
k8s/backend/
├── kustomization.yml
├── deployment.yml
└── service.yml
```

`deployment.yml`에는 기본 image가 들어 있고, 실제 배포 시에는 `kustomization.yml`의 `images.newTag`를 GitHub Actions가 변경합니다.

예시:

```yaml
images:
  - name: amdp-registry.skala-ai.com/skala3-finalproj-class2-team8/skipa-backend
    newTag: dev-a1b2c3d
```

이 방식으로 image tag 변경을 `deployment.yml`이 아니라 `kustomization.yml`에 집중시킵니다.

## 이미지 태그 전략

`dev-latest`만 사용하면 Git manifest가 바뀌지 않아 Argo CD가 변경을 감지하지 못할 수 있습니다.
따라서 각 서비스는 고유 태그를 사용합니다.

```text
skipa-backend:dev-<short-sha>
skipa-frontend:dev-<short-sha>
skipa-ai:dev-<short-sha>
```

예시:

```text
skipa-backend:dev-a1b2c3d
```

서비스 레포의 GitHub Actions는 다음 두 태그를 함께 push할 수 있습니다.

```text
dev-<short-sha>  # 실제 배포에 사용
dev-latest       # 참고용 또는 수동 확인용
```

실제 Argo CD 배포 기준은 `kustomization.yml`에 기록된 `dev-<short-sha>` 태그입니다.

## 배포 흐름

각 서비스 레포에서 main 브랜치에 merge되면 GitHub Actions가 실행됩니다.
GitHub Actions는 이미지를 빌드하고 Harbor에 push한 뒤, 이 레포의 해당 서비스 `kustomization.yml` image tag를 업데이트합니다.
Argo CD는 이 레포의 변경을 감지하여 EKS 클러스터에 자동으로 sync합니다.

Backend 예시:

```text
skipa-backend main merge
→ GitHub Actions 실행 (skipa-backend 레포)
→ Spring Boot jar 빌드
→ Docker image 빌드
→ Harbor push
→ skipa-infra/k8s/backend/kustomization.yml newTag 수정 후 commit
→ Argo CD가 skipa-infra 변경 감지
→ team8-skipa-backend Application sync
→ RollingUpdate로 backend Pod 교체
```

Frontend, AI도 동일한 방식으로 각각 자기 디렉토리의 `kustomization.yml`만 수정합니다.

```text
skipa-frontend → k8s/frontend/kustomization.yml
skipa-backend  → k8s/backend/kustomization.yml
skipa-ai       → k8s/ai/kustomization.yml
```

## GitHub Secrets

각 서비스 레포의 GitHub Actions에서 Harbor에 이미지를 push하고 이 레포에 commit하기 위해 아래 secret이 필요합니다.

```text
HARBOR_USERNAME
HARBOR_PASSWORD
INFRA_REPO_TOKEN        # skipa-infra 레포에 push하기 위한 PAT
```

등록 위치:

```text
GitHub Repository
→ Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

## 백엔드 환경변수

백엔드 Deployment는 아래 환경변수를 사용합니다.

```text
SPRING_PROFILES_ACTIVE=prod

DB_HOST=skipa-postgres
DB_PORT=5432
DB_NAME=skipa
DB_USERNAME=skipa
DB_PASSWORD=<PostgreSQL Secret password>

REDIS_HOST=skipa-redis
REDIS_PORT=6379
REDIS_PASSWORD=<Redis Secret password>

JWT_SECRET=<Backend JWT Secret>
```

## 로컬에서 DB 확인

PostgreSQL은 외부에 직접 공개하지 않습니다.
DataGrip 등 로컬 클라이언트에서 확인할 때는 `port-forward`를 사용합니다.

```bash
kubectl port-forward svc/skipa-postgres 15432:5432
```

DataGrip 연결 정보:

```text
Host: 127.0.0.1
Port: 15432
Database: skipa
User: skipa
Password: skipa-postgres-secret의 password 값
```

Redis 확인:

```bash
kubectl port-forward svc/skipa-redis 16379:6379
```

```bash
redis-cli -h 127.0.0.1 -p 16379 -a '<REDIS_PASSWORD>' PING
```

## 주의사항

- Secret 값은 GitHub에 커밋하지 않습니다.
- `.env` 파일은 GitHub에 커밋하지 않습니다.
- Harbor 비밀번호, JWT Secret은 코드에 직접 작성하지 않습니다.
- PostgreSQL/Redis는 외부에 직접 공개하지 않고 `ClusterIP`로 유지합니다.
- PVC를 삭제하면 연결된 실제 볼륨 데이터도 삭제될 수 있으므로 주의합니다.
- `gp3` StorageClass의 ReclaimPolicy가 `Delete`이므로 PVC 삭제 시 데이터가 함께 삭제될 수 있습니다.
- Datastore는 Argo CD로 관리하지만, 서비스 배포 자동화 대상에서는 제외합니다.
- frontend/backend/ai 배포 시에는 각 서비스의 `kustomization.yml` image tag만 변경합니다.