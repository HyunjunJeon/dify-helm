# dify-helm-custom

## 커스터마이징 설치

`helm install`/`helm upgrade` 명령어에 `-f` 옵션을 사용하여 자신만의 `values.yaml`을 적용하세요.
방대한 내용이지만 아래 섹션별로 정리되어 있으니 걱정하지 마세요:

1. 이미지: 모든 Dify 컴포넌트의 이미지 조정
2. 클라우드별 커스텀 이미지: Azure ACR, AWS ECR, GCP GCR 통합 가이드
3. Dify 서비스: 각 Dify 컴포넌트의 구성 커스터마이징
4. 미들웨어: 내장 미들웨어의 구성 설정
5. 외부 서비스: 내장 데이터 지속성을 위한 외부 서비스 대체

### 1. 이미지 조정

다양한 컴포넌트에 대해 커스텀 이미지를 지정할 수 있습니다:

```yaml
# values.yaml
images:
  api:
    repository: your-registry/dify-api
    tag: your-tag
    pullPolicy: IfNotPresent
  worker:
    repository: your-registry/dify-worker
    tag: your-tag
    pullPolicy: IfNotPresent
  sandbox:
    repository: your-registry/dify-sandbox
```

### 2. 클라우드별 커스텀 이미지 가이드

#### Azure Container Registry (ACR) 커스텀 이미지

Azure 환경에서 ACR의 커스텀 이미지를 사용하는 경우:

```yaml
# values-azure.yaml
image:
  api:
    repository: "yourregistry.azurecr.io/dify/api"
    tag: "v1.0.0"
    pullPolicy: "Always"
  web:
    repository: "yourregistry.azurecr.io/dify/web"
    tag: "v1.0.0"
    pullPolicy: "Always"
  # worker는 api와 같은 이미지 사용
  worker:
    repository: "yourregistry.azurecr.io/dify/api"
    tag: "v1.0.0"
    pullPolicy: "Always"

# ACR 인증 설정 (필요시)
imagePullSecrets:
  - name: acr-secret
```

#### ACR 인증 설정 방법

**방법 1: AKS-ACR 통합 (권장)**

```bash
az aks update -n your-cluster -g your-resource-group --attach-acr yourregistry
```

**방법 2: Docker Registry Secret 생성**

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=yourregistry.azurecr.io \
  --docker-username=<service-principal-id> \
  --docker-password=<service-principal-password> \
  --namespace=dify
```

#### 커스텀 이미지 요구사항

1. **API 이미지**:
   - 기존 Dify API와 호환되는 Flask/FastAPI 애플리케이션
   - 포트 5001에서 서비스 제공
   - 환경변수 기반 설정 지원

2. **Web 이미지**:
   - 기존 Dify Web과 호환되는 Next.js 애플리케이션
   - 포트 3000에서 서비스 제공
   - API 엔드포인트 연결 설정

3. **환경변수 호환성**:
   - 데이터베이스 연결 정보 (POSTGRES_*, REDIS_*)
   - Dify 설정 변수 (SECRET_KEY, CHECK_UPDATE_URL 등)
   - 스토리지 설정 (S3_*, AZURE_STORAGE_* 등)

#### 배포 및 롤백

**커스텀 이미지로 배포:**

```bash
helm upgrade dify ./charts/dify --namespace dify -f charts/dify/values-azure.yaml
```

**원본 이미지로 롤백:**

```bash
# values-azure.yaml에서 image 섹션 주석 처리 후
helm upgrade dify ./charts/dify --namespace dify -f charts/dify/values-azure.yaml
```

#### AWS Elastic Container Registry (ECR) 커스텀 이미지

```yaml
# values-aws.yaml
image:
  api:
    repository: "123456789012.dkr.ecr.us-west-2.amazonaws.com/dify/api"
    tag: "v1.0.0"
    pullPolicy: "Always"
  web:
    repository: "123456789012.dkr.ecr.us-west-2.amazonaws.com/dify/web"
    tag: "v1.0.0"
    pullPolicy: "Always"

# ECR 인증 설정 (EKS에서 자동 처리되지만 필요시)
imagePullSecrets:
  - name: ecr-secret
```

**ECR 인증 설정:**

```bash
# AWS CLI 설정 후
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-west-2.amazonaws.com

# EKS에서는 보통 자동으로 처리됨 (IAM 역할 기반)
```

#### Google Container Registry (GCR) 커스텀 이미지

```yaml
# values-gcp.yaml
image:
  api:
    repository: "gcr.io/your-project-id/dify/api"
    tag: "v1.0.0"
    pullPolicy: "Always"
  web:
    repository: "gcr.io/your-project-id/dify/web"
    tag: "v1.0.0"
    pullPolicy: "Always"

# GCR 인증 설정
imagePullSecrets:
  - name: gcr-secret
```

**GCR 인증 설정:**

```bash
# Service Account Key 사용
kubectl create secret docker-registry gcr-secret \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat key.json)" \
  --namespace=dify

# 또는 GKE Workload Identity 사용 (권장)
```

### 3. Dify 컴포넌트 커스터마이징

#### 데이터 지속성

내장 데이터 지속성을 커스터마이징하려면, `values.yaml`의 `persistence` 섹션에서 `enabled: true`로 설정하고 스토리지 클래스와 크기를 지정하세요. 예를 들어:

```yaml
# values.yaml
api:
  persistence:
    enabled: true
    storageClass: your-storage-class
    accessMode: ReadWriteMany
    size: 10Gi

```

또는 기존의 `PersistentVolumeClaim`을 지정할 수 있습니다:

```yaml
# values.yaml
api:
  persistence:
    enabled: true
    persistentVolumeClaim:
      existingClaim: "your-pvc-name"
```

#### 환경 변수

이 차트는 데이터 지속성, 서비스 디스커버리, 데이터베이스 연결 등을 위한 환경 변수를 자동으로 관리합니다. 추가 환경 변수를 적용하거나 기존 환경 변수를 재정의하려면, 각 컴포넌트의 `extraEnv` 섹션을 참조하세요:

```yaml
# values.yaml
...
api:
  extraEnv:
  # 직접 값 설정 방법
  - name: LANG
    value: "C.UTF-8"
  # 기존 ConfigMap 사용
  - name: MY_CONFIG
    valueFrom:
      configMapKeyRef:
        name: my-config
        key: MY_CONFIG
  # 기존 Secret 사용
  - name: MY_SECRET
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: MY_SECRET
```

### 4. 내장 미들웨어 작업

내장된 `Redis`, `PostgreSQL`, `weaviate`는 사용자가 빠른 시작을 위해 자체 포함된 `Dify` 환경을 구성할 수 있도록 합니다. 이러한 컴포넌트들은 서드파티 헬름 차트에서 제공됩니다. 내장 미들웨어를 커스터마이징하려면, 섹션 이름과 공식 문서를 참조하세요:

| 섹션 | 문서 |
| ----- | --- |
| `redis` | [bitnami/redis](https://github.com/bitnami/charts/tree/main/bitnami/redis) |
| `postgresql` | [bitnami/postgresql](https://github.com/bitnami/charts/tree/main/bitnami/postgresql) |
| `weaviate` | [weaviate](https://github.com/weaviate/weaviate-helm) |

이들을 비활성화하려면, `values.yaml`의 해당 섹션에서 `enabled: false`로 설정하고 외부 서비스 제공자를 적용하세요:

```yaml
# values.yaml
redis:
  enabled: false  # 내장 Redis 비활성화
```

### 5. 외부 서비스 선택

프로덕션 환경에서는 내장 미들웨어보다 엔터프라이즈급 제공업체의 서비스를 사용하는 것이 권장됩니다. 예를 들어, 내장 `Redis`를 대체하려면:

```yaml
# values.yaml
externalRedis:
  enabled: true
  host: "redis.example"
  port: 6379
  username: ""
  password: "difyai123456"
  useSSL: false
```

자세한 내용은 `external<Service>` 섹션을 참조하세요.
