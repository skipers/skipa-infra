# skipa-infra

이 레포지토리는 SKIPA 서비스의 Kubernetes manifest 및 Argo CD 설정을 관리합니다.  
CI는 각 서비스 레포의 GitHub Actions가 담당하고, CD는 이 레포를 기반으로 Argo CD가 담당합니다.

## 디렉토리 구조

```text
skipa-infra/
├── argocd/
│   ├── root/
│   │   └── skipa-root-application.yml
│   └── apps/
│       ├── backend-application.yml
│       ├── frontend-application.yml
│       ├── ai-application.yml
│       └── datastore-application.yml
└── k8s/
    ├── backend/
    │   ├── deployment.yml
    │   └── service.yml
    ├── frontend/
    │   ├── deployment.yml
    │   └── service.yml
    ├── ai/
    │   ├── deployment.yml
    │   └── service.yml
    └── datastore/
        ├── postgres-service.yml
        ├── postgres-statefulset.yml
        ├── redis-service.yml
        └── redis-statefulset.yml
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
- Image Registry: Harbor
- Image: `amdp-registry.skala-ai.com/skala3-finalproj-class2-team8/skipa-frontend:<tag>`

### AI

- Deployment: `skipa-ai`
- Service: `skipa-ai`
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

## Argo CD 설정

### App of Apps 구조

root Application이 `argocd/apps/` 디렉토리를 바라보며, 하위 Application들을 자동으로 생성 및 관리합니다.

```text
team8-skipa (root)
├── team8-skipa-backend
├── team8-skipa-frontend
├── team8-skipa-ai
└── team8-skipa-datastore
```

### root Application 생성

root Application 하나만 수동으로 적용합니다.  
이후 `argocd/apps/` 디렉토리의 변경은 Argo CD가 자동으로 감지합니다.

```bash
kubectl apply -f argocd/root/skipa-root-application.yml
```

확인:

```bash
kubectl get application -n skala-argocd
kubectl describe application team8-skipa -n skala-argocd
```

## Datastore 수동 적용

Datastore는 초기 세팅 시 수동으로 적용합니다.

```bash
kubectl apply -f k8s/datastore/postgres-service.yml
kubectl apply -f k8s/datastore/postgres-statefulset.yml

kubectl apply -f k8s/datastore/redis-service.yml
kubectl apply -f k8s/datastore/redis-statefulset.yml
```

상태 확인:

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

## 배포 흐름

각 서비스 레포에서 main 브랜치에 merge되면 GitHub Actions가 실행됩니다.  
GitHub Actions는 이미지를 빌드하고 Harbor에 push한 뒤, 이 레포의 해당 `deployment.yml` image tag를 업데이트합니다.  
Argo CD는 이 레포의 변경을 감지하여 EKS 클러스터에 자동으로 sync합니다.

```text
skipa-backend main merge
→ GitHub Actions 실행 (skipa-backend 레포)
→ Spring Boot jar 빌드
→ Docker image 빌드
→ Harbor push
→ skipa-infra/k8s/backend/deployment.yml image tag 수정 후 commit
→ Argo CD가 skipa-infra 변경 감지
→ EKS에 자동 sync
→ RollingUpdate로 백엔드 Pod 교체
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
- Datastore 리소스는 서비스 배포와 분리해서 관리합니다.
