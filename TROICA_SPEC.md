# Troica Market Service MSA — Polyrepo CI/CD Implementation Blueprint

> **목적**: 이 문서는 Claude Code가 KT Cloud Tech UP 2기 Troica 팀의 Market Service MSA 프로젝트를 polyrepo 구조로 implementation할 때 따라야 할 단일 진입점 청사진이다. 모든 결정·구조·코드 스니펫은 자체 검증되었으며, 계획서(`Market_Service_MSA_구축_프로젝트_제안_및_계획서_2차수정본.pdf`)와 발표자료를 기준으로 작성되었다.
>
> **작업 원칙**: 이 문서의 모든 코드 블록은 그대로 사용 가능한 골격이다. 다만 placeholder(`<...>`, `${{ secrets.* }}`, `ACCOUNT_ID` 등)는 실제 값으로 치환 필요. 검증 안 된 추정은 명시했다.

---

## 0. 결정 사항 요약 (Team Decisions)

| 항목 | 결정 |
|------|------|
| 레포지토리 전략 | Polyrepo (9개) — 서비스별 + 라이브러리 + manifest + provisioning |
| 이미지 레지스트리 | **AWS ECR + GitHub OIDC** (long-lived AWS key 미사용) |
| 매니페스트 레포 구조 | `platform/` + `applications/` + `projects/` (계획서 E2) |
| 마이크로서비스 패키징 | **공통 Helm 차트 1개 + values per service** (계획서 5.5.3 의도) |
| 환경 분기 | `dev` (CI push 시 자동 머지 → 즉시 CD) + `prod` (PR 머지 시 CD) |
| 이미지 빌드 | **Multistage Dockerfile + Trivy 스캔** |
| order-worker | 별도 레포 아님 — order-service 코드베이스 내 `SPRING_PROFILES_ACTIVE=worker` 분기 + 별도 K8s Deployment |
| 알림 라우팅 | **AlertManager → PagerDuty (critical) + Slack (warning/info)** — Grafana Alerting 아님 |
| Notification 서비스 | Kafka 컨슈머 (`order.confirmed`, `order.cancelled`, `notification.requested`) + SES + Slack Webhook 멀티채널 |
| payment / shipping | 별도 서비스 없음, order-service 내 상태 변경으로 시뮬레이션 |
| GitOps 패턴 | App-of-Apps + Sync Wave (플랫폼) + ApplicationSet Git Generator (마이크로서비스) |
| 이벤트 스키마 | **Protobuf** (Avro 미사용). PoC 단계는 schema registry 없이 `msa-common-libs:events` JAR로 클래스 공유. 호환은 SemVer + 필드번호 불변 규칙으로 강제 |
| Service mesh / Ingress | **Istio** (Traefik 아님). 현 `msa-provisioning`의 Traefik은 Phase 5에서 제거 |
| 빌드 toolchain | **Spring Boot 3.5.13** + **Kotlin 2.1.0** + **Gradle 8.10.2** + **JVM 21** 통일. SB 3.5는 2026-06-30 OSS EOL — Phase 6 후 3.6/4.0 마이그레이션 검토. Kotlin 2.3.x + Gradle 9.x는 `BuildUtilKt.clearJarCaches()` JetBrains 버그로 보류 |
| Spring Cloud (Phase 4 api-gateway) | **Spring Cloud 2025.0.2 (Northfields)** — SB 3.5.13 공식 짝. Gateway WebFlux는 `spring-cloud-gateway-server-webflux` 명시 아티팩트 사용 (legacy `spring-cloud-starter-gateway` 도 동작하나 향후 제거 예정) |

---

## 1. 레포지토리 구조 (9개)

GitHub Organization: `KTCloud-CloudNative-Troica-Team`

### 1.1 애플리케이션 코드 (6개)

각 레포는 독립적인 GitHub Actions CI 워크플로우, Dockerfile, Helm 차트 reference 보유.

| # | 레포 명 | 책임 (계획서 2.2 도메인 경계) |
|---|---------|----------------------------|
| 1 | `msa-user-service` | 회원 가입·인증·JWT 발급 |
| 2 | `msa-product-service` | 상품 정보·카테고리 관리 |
| 3 | `msa-inventory-service` | 재고 조회·차감·이벤트 소싱 (Event Sourcing) |
| 4 | `msa-order-service` | 주문 생성·상태 관리·Saga 오케스트레이션 (+ order-worker 분기 Deployment) |
| 5 | `msa-notification-service` | 이벤트 기반 알림 (SES + Slack 멀티채널) |
| 6 | `msa-api-gateway` | Spring Cloud Gateway, JWT 필터, Rate Limit |

### 1.2 공용 라이브러리 (1개)

| # | 레포 명 | 내용 |
|---|---------|------|
| 7 | `msa-common-libs` | **멀티모듈 Gradle** 프로젝트. 4개 서브모듈: `common` (JPA/QueryDSL/예외/유틸), `client-redis` (분산락+멱등 aspect), `client-ses` (SES 발송), `events` (Protobuf 5개 스키마 + codegen). GitHub Packages로 모듈별 publish |

### 1.3 GitOps & 인프라 (2개, 기존)

| # | 레포 명 | 내용 |
|---|---------|------|
| 8 | `msa-argocd-manifest` | Helm 차트 + values + ArgoCD Application/ApplicationSet/Project CRD |
| 9 | `msa-provisioning` | Terraform (AWS) + Ansible (kubeadm) |

> **archive 처리**: 기존 `msa-spring-boot` 모노레포는 마이그레이션 완료 후 read-only로 보존 (delete 금지, 이력 보존용).

---

## 2. 매니페스트 레포 디렉토리 구조 (`msa-argocd-manifest`)

```
msa-argocd-manifest/
├── README.md
├── bootstrap/
│   └── root-app.yaml                       # STEP 3 kubectl apply 대상 — App-of-Apps의 root
│
├── projects/                               # ArgoCD AppProject CRD
│   ├── platform-project.yaml
│   └── market-project.yaml
│
├── applications/                           # 마이크로서비스 레이어 (ApplicationSet)
│   ├── appset.yaml                         # Matrix generator: services × envs
│   ├── charts/
│   │   └── microservice/                   # 공통 Helm 차트 (모든 서비스 공유)
│   │       ├── Chart.yaml
│   │       ├── values.yaml                 # 차트 기본값
│   │       └── templates/
│   │           ├── _helpers.tpl
│   │           ├── deployment.yaml
│   │           ├── service.yaml
│   │           ├── hpa.yaml
│   │           ├── serviceaccount.yaml
│   │           ├── configmap.yaml
│   │           └── virtualservice.yaml     # Istio
│   └── values/                             # 서비스별 + 환경별 values
│       ├── user-service/
│       │   ├── values.yaml                 # 서비스 공통값
│       │   ├── values-dev.yaml
│       │   └── values-prod.yaml
│       ├── product-service/
│       ├── inventory-service/
│       ├── order-service/
│       │   ├── values.yaml                 # worker Deployment는 .Values.worker.enabled 플래그로 분기 (별도 values 파일 없음)
│       │   ├── values-dev.yaml
│       │   └── values-prod.yaml
│       ├── notification-service/
│       └── api-gateway/
│
└── platform/                               # 플랫폼 레이어 (App-of-Apps + Sync Wave)
    ├── root.yaml                           # platform Application들의 root
    ├── 00-cert-manager/
    │   └── application.yaml                # syncWave: -10
    ├── 10-istio-base/
    │   └── application.yaml                # syncWave: -5
    ├── 11-istio-cp/
    │   └── application.yaml                # syncWave: -4
    ├── 30-kube-prometheus-stack/           # Prometheus + AlertManager + Grafana
    │   ├── application.yaml
    │   └── values.yaml
    ├── 30-loki/
    ├── 30-tempo/
    ├── 30-mimir/
    ├── 40-strimzi-operator/                # Kafka operator
    ├── 40-cnpg-operator/                   # PostgreSQL operator
    ├── 40-redis-operator/
    ├── 50-kafka-cluster/                   # Strimzi Kafka CR + KafkaTopic CRs
    │   ├── kafka.yaml
    │   └── topics/
    │       ├── order-pending.yaml
    │       ├── order-inventory-reserved.yaml
    │       ├── order-confirmed.yaml
    │       ├── order-cancelled.yaml
    │       └── notification-requested.yaml
    ├── 50-postgres-clusters/               # CNPG Cluster CRs (서비스별)
    └── 50-redis-cluster/
```

> **Sync Wave 규칙**: 음수일수록 먼저 실행. CRD → operator → 컴포넌트 → 의존 컴포넌트 순. 폴더명 prefix는 시각적 가이드이고 실제 순서는 Application CRD의 `argocd.argoproj.io/sync-wave` annotation으로 강제.

---

## 3. 공통 Helm 차트 (`applications/charts/microservice/`)

### 3.1 Chart.yaml

```yaml
apiVersion: v2
name: microservice
description: Troica common chart for Spring Boot microservices
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### 3.2 values.yaml (차트 기본값)

```yaml
nameOverride: ""
fullnameOverride: ""

image:
  repository: ""              # values per service에서 override
  tag: "latest"               # CI가 main-<sha>로 override
  pullPolicy: IfNotPresent

replicaCount: 2

serviceAccount:
  create: true
  annotations: {}

service:
  type: ClusterIP
  port: 8080

# Spring Boot 프로파일 — order-worker 같은 분기에 사용
springProfilesActive: ""      # 예: "dev,worker"

# 환경변수 — values per service/env에서 추가
env: []
# - name: SPRING_DATASOURCE_URL
#   value: jdbc:postgresql://...

envFrom: []
# - secretRef:
#     name: order-service-secrets

resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi

hpa:
  enabled: false
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 10

istio:
  enabled: true
  virtualService:
    enabled: false
    hosts: []
    gateways: []

# order-worker처럼 동일 이미지로 별도 Deployment를 추가로 띄울 때
# values-worker.yaml에서 이 섹션을 활성화
worker:
  enabled: false
  springProfilesActive: "worker"
  replicaCount: 1
  command: []                 # 필요시 entrypoint 오버라이드
```

### 3.3 templates/_helpers.tpl

```yaml
{{- define "microservice.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "microservice.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- printf "%s" $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}

{{- define "microservice.labels" -}}
app.kubernetes.io/name: {{ include "microservice.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/version: {{ .Values.image.tag | quote }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
{{- end -}}

{{- define "microservice.selectorLabels" -}}
app.kubernetes.io/name: {{ include "microservice.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

### 3.4 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "microservice.fullname" . }}
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "microservice.selectorLabels" . | nindent 6 }}
      role: api
  template:
    metadata:
      labels:
        {{- include "microservice.selectorLabels" . | nindent 8 }}
        role: api
      annotations:
        sidecar.istio.io/inject: "{{ .Values.istio.enabled }}"
    spec:
      serviceAccountName: {{ include "microservice.fullname" . }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
          env:
            {{- if .Values.springProfilesActive }}
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.springProfilesActive | quote }}
            {{- end }}
            {{- with .Values.env }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}

{{- if .Values.worker.enabled }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "microservice.fullname" . }}-worker
  labels:
    {{- include "microservice.labels" . | nindent 4 }}
    role: worker
spec:
  replicas: {{ .Values.worker.replicaCount }}
  selector:
    matchLabels:
      {{- include "microservice.selectorLabels" . | nindent 6 }}
      role: worker
  template:
    metadata:
      labels:
        {{- include "microservice.selectorLabels" . | nindent 8 }}
        role: worker
      annotations:
        sidecar.istio.io/inject: "{{ .Values.istio.enabled }}"
    spec:
      serviceAccountName: {{ include "microservice.fullname" . }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.worker.springProfilesActive | quote }}
            {{- with .Values.env }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
{{- end }}
```

### 3.5 templates/service.yaml, templates/hpa.yaml, templates/serviceaccount.yaml

> **참고**: Claude Code가 표준 패턴으로 채울 수 있는 보일러플레이트. `helm create`로 골격 받아서 위 deployment 패턴에 맞게 정리하면 됨. selectorLabels에 `role: api`만 매칭하도록 service의 selector 조정 (worker pod는 service에 포함 안 됨).

---

## 4. values per service 예시

### 4.1 `applications/values/order-service/values.yaml` (공통)

```yaml
nameOverride: order-service
image:
  repository: <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/msa/order-service
  # tag는 CI가 환경별 values-{env}.yaml에서 override

service:
  port: 8080

envFrom:
  - secretRef:
      name: order-service-secrets         # ExternalSecrets로 주입
  - configMapRef:
      name: order-service-config

istio:
  enabled: true
```

### 4.2 `applications/values/order-service/values-dev.yaml`

```yaml
image:
  tag: main-PLACEHOLDER                   # CI가 yq로 교체
replicaCount: 1
springProfilesActive: "dev"
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 300m
    memory: 512Mi

# Outbox Poller worker도 같이 띄움
worker:
  enabled: true
  springProfilesActive: "dev,worker"
  replicaCount: 1
```

### 4.3 `applications/values/order-service/values-prod.yaml`

```yaml
image:
  tag: main-PLACEHOLDER                   # CI가 yq로 교체 → PR로 머지
replicaCount: 3
springProfilesActive: "prod"
hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi

worker:
  enabled: true
  springProfilesActive: "prod,worker"
  replicaCount: 2
```

---

## 5. ApplicationSet (마이크로서비스 자동 등록)

`applications/appset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: market-services
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest.git
              revision: main
              directories:
                - path: applications/values/*
          - list:
              elements:
                - env: dev
                - env: prod
  template:
    metadata:
      name: '{{.path.basename}}-{{.env}}'
      labels:
        env: '{{.env}}'
    spec:
      project: market
      source:
        repoURL: https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest.git
        targetRevision: main
        path: applications/charts/microservice
        helm:
          valueFiles:
            - ../../values/{{.path.basename}}/values.yaml
            - ../../values/{{.path.basename}}/values-{{.env}}.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: market-{{.env}}
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

> **검증 포인트**: `applications/values/*` 디렉토리만 추가하면 ApplicationSet이 자동으로 `{서비스명}-dev`, `{서비스명}-prod` 두 Application을 생성한다. 새 서비스 추가 시 ArgoCD 측 작업 0.

---

## 6. App-of-Apps for platform layer

`platform/root.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest.git
    targetRevision: main
    path: platform
    directory:
      recurse: true
      include: '{*/application.yaml,*/*/application.yaml}'
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

각 컴포넌트 (`platform/30-kube-prometheus-stack/application.yaml` 예시):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "30"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  # ArgoCD multi-source 패턴: 외부 chart + git repo의 values를 합성.
  # `source:` (단수)와 `sources:` (복수)는 상호배타이므로 multi-source는 반드시 `sources:`만 사용한다.
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 65.x.x                # 실제 최신 stable 버전은 apply 직전에 검증
      helm:
        valueFiles:
          - $values/platform/30-kube-prometheus-stack/values.yaml
    - repoURL: https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

> **주의**: ArgoCD multi-source application은 v2.6+ 필요. 단일 source로 가려면 chart 자체를 git 레포에 vendoring하거나 Application별로 values 인라인.

---

## 7. CI 워크플로우 (각 서비스 레포 공통)

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write          # OIDC
  contents: read
  pull-requests: write     # prod PR 생성
  packages: write          # GitHub Packages (common-libs pull)

env:
  AWS_REGION: ap-northeast-2
  SERVICE_NAME: order-service              # 각 레포마다 수정
  ECR_REPO: msa/order-service              # 각 레포마다 수정
  MANIFEST_REPO: KTCloud-CloudNative-Troica-Team/msa-argocd-manifest

jobs:
  build-test:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: gradle

      - uses: gradle/actions/setup-gradle@v4

      - name: Build & test
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}   # GitHub Packages 인증
        run: ./gradlew build test --no-daemon

      - name: Set image tag
        id: meta
        run: echo "tag=main-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: Configure AWS credentials (OIDC)
        if: github.event_name == 'push'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/troica-gha-ecr-push
          aws-region: ${{ env.AWS_REGION }}

      - name: ECR login
        if: github.event_name == 'push'
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        if: github.event_name == 'push'
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          load: true
          tags: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO }}:${{ steps.meta.outputs.tag }}

      # 주의: 2026-03-19 aquasecurity/trivy-action 공급망 사고로 0.0.1~0.34.2 태그는 모두 compromised.
      # 사고 후 새 태그 컨벤션은 `v` prefix. v0.35.0 부터 안전. 본 PoC는 v0.36.0(2026-04-22) 핀.
      # 보안 강화 시 commit SHA 핀 + Dependabot/Renovate 도입 권장.
      - name: Trivy scan
        if: github.event_name == 'push'
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO }}:${{ steps.meta.outputs.tag }}
          format: table
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true

      - name: Push to ECR
        if: github.event_name == 'push'
        run: |
          docker push ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO }}:${{ steps.meta.outputs.tag }}

  update-dev:
    needs: build-test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: ${{ env.MANIFEST_REPO }}
          token: ${{ secrets.MANIFEST_PAT }}

      - uses: mikefarah/yq@v4

      - name: Bump dev image tag
        run: |
          yq -i '.image.tag = "${{ needs.build-test.outputs.image-tag }}"' \
            applications/values/${{ env.SERVICE_NAME }}/values-dev.yaml

      - name: Commit & push to main (dev = auto)
        run: |
          git config user.name "troica-ci-bot"
          git config user.email "ci-bot@troica"
          git add applications/values/${{ env.SERVICE_NAME }}/values-dev.yaml
          git commit -m "deploy(${{ env.SERVICE_NAME }}/dev): ${{ needs.build-test.outputs.image-tag }}"
          git push origin main

  update-prod:
    needs: build-test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: ${{ env.MANIFEST_REPO }}
          token: ${{ secrets.MANIFEST_PAT }}

      - uses: mikefarah/yq@v4

      - name: Bump prod image tag
        run: |
          yq -i '.image.tag = "${{ needs.build-test.outputs.image-tag }}"' \
            applications/values/${{ env.SERVICE_NAME }}/values-prod.yaml

      - name: Create PR for prod
        uses: peter-evans/create-pull-request@v7
        with:
          token: ${{ secrets.MANIFEST_PAT }}
          branch: bump/${{ env.SERVICE_NAME }}-prod-${{ needs.build-test.outputs.image-tag }}
          base: main
          title: "deploy(${{ env.SERVICE_NAME }}/prod): ${{ needs.build-test.outputs.image-tag }}"
          commit-message: "deploy(${{ env.SERVICE_NAME }}/prod): ${{ needs.build-test.outputs.image-tag }}"
          body: |
            Auto-generated prod deployment PR.

            **Service**: ${{ env.SERVICE_NAME }}
            **Image tag**: `${{ needs.build-test.outputs.image-tag }}`
            **Source commit**: ${{ github.event.compare }}

            Manual review required. Merge to deploy to prod.
          labels: |
            prod-deploy
            ${{ env.SERVICE_NAME }}
```

> **dev 자동 / prod 수동의 핵심**: `update-dev`는 main에 직접 push (즉시 ArgoCD sync), `update-prod`는 별도 브랜치 + PR 생성 (사람이 머지해야 sync). 동일 main 브랜치를 ArgoCD가 watch하므로 dev/prod ApplicationSet은 같은 commit history에서 다른 values 파일을 본다.

### 7.1 GitHub Secrets에 설정해야 할 값

| Secret 명 | 용도 | 범위 |
|----------|------|------|
| `AWS_ACCOUNT_ID` | OIDC role ARN 조립 | Organization |
| `MANIFEST_PAT` | manifest repo write 권한 PAT (fine-grained, `msa-argocd-manifest` contents:write + pull-requests:write) | Organization |

`GITHUB_TOKEN`은 자동 발급, 별도 등록 불필요.

---

## 8. Multistage Dockerfile (각 서비스 레포 공통 패턴)

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7

# ===== Stage 1: build =====
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Gradle wrapper와 의존성 메타만 먼저 복사 (캐시 최적화)
COPY gradlew settings.gradle.kts build.gradle.kts ./
COPY gradle gradle
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon || true

# 소스 복사 후 빌드
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# ===== Stage 2: extract layers (Spring Boot layered jar) =====
FROM eclipse-temurin:21-jre-alpine AS layers
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ===== Stage 3: runtime =====
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
WORKDIR /app

# 레이어 순서: 변경 빈도 낮은 순
COPY --from=layers /app/dependencies/ ./
COPY --from=layers /app/spring-boot-loader/ ./
COPY --from=layers /app/snapshot-dependencies/ ./
COPY --from=layers /app/application/ ./

EXPOSE 8080
ENV JAVA_OPTS="-XX:+UseG1GC -XX:MaxRAMPercentage=75.0"
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

> **Spring Boot 3.5.x 검증 포인트**: 클래스 경로가 `org.springframework.boot.loader.launch.JarLauncher` (3.2부터 변경). 3.2 미만이라면 `org.springframework.boot.loader.JarLauncher`. 빌드 후 `unzip -l build/libs/*.jar | grep Launcher` 로 실제 경로 확인 권장.

> **`.dockerignore` 필수**:
> ```
> .gradle
> build
> .git
> .idea
> *.iml
> .env
> ```

---

## 9. Notification Service 상세 설계

### 9.1 토픽 매핑 (계획서 부록 A 기반)

| 토픽 | notification-service 동작 | 처리 후 액션 |
|------|------------------------|-------------|
| `order.confirmed` | 구독 | 주문자 이메일 (SES) + 운영 채널 Slack |
| `order.cancelled` | 구독 | 주문자 이메일 (SES) + 운영 채널 Slack |
| `notification.requested` | 구독 | payload의 `channel`에 따라 분기 (EMAIL/SLACK/SMS) |

### 9.2 KafkaTopic CRD (Strimzi)

`platform/50-kafka-cluster/topics/order-confirmed.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order.confirmed
  namespace: kafka
  labels:
    strimzi.io/cluster: market-kafka
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 604800000          # 7일
    segment.bytes: 1073741824        # 1GB
    min.insync.replicas: 2
    cleanup.policy: delete
```

5개 토픽 모두 동일 패턴. `notification.requested`만 retention을 짧게(예: 1일) 가져가도 됨 — 즉시 처리 의도.

### 9.3 페이로드 스키마 — Protobuf

**결정 (Phase 2)**: 모든 토픽 페이로드는 Protobuf. `msa-common-libs/events` 모듈에 `.proto` 파일을 두고 codegen된 Java/Kotlin 클래스를 모든 생산자/소비자가 공유. **PoC 단계에서 schema registry는 도입하지 않음** — 단일 라이브러리 + SemVer로 호환 관리. 추후 멀티팀 거버넌스 필요 시 Apicurio Registry 추가 가능.

선정 근거: ① 모노레포가 이미 gRPC + Protobuf 사용 중 (도구·지식 재사용), ② 2026년 기준 Confluent/Apicurio Registry 모두 Protobuf first-class 지원, ③ Avro의 self-describing 우위가 Kafka 마이크로서비스 시나리오에서는 codegen 타입 안전성으로 상쇄, ④ CNCF 생태계 표준 (Kubernetes API, Istio, OpenTelemetry).

#### 5개 토픽 ↔ Proto 메시지 매핑

| Kafka topic | Proto message | 흐름 |
|-------------|---------------|------|
| `order.pending`              | `troica.order.v1.OrderPending`              | order(Outbox poller) → inventory |
| `order.inventory-reserved`   | `troica.order.v1.OrderInventoryReserved`    | inventory → order |
| `order.confirmed`            | `troica.order.v1.OrderConfirmed`            | order → notification |
| `order.cancelled`            | `troica.order.v1.OrderCancelled`            | order → notification, inventory(보상) |
| `notification.requested`     | `troica.notification.v1.NotificationRequested` | any → notification |

#### `notification.requested` proto (요지)

```protobuf
syntax = "proto3";
package troica.notification.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

message NotificationRequested {
  string idempotency_key = 1;
  Channel channel = 2;
  string template = 3;
  Recipient recipient = 4;
  google.protobuf.Struct payload = 5;   // 자유형 변수 (template 렌더링용)
  google.protobuf.Timestamp occurred_at = 6;
}

message Recipient {
  string user_id = 1;
  string email = 2;
  string slack_user_id = 3;
  string phone_number = 4;
}

enum Channel {
  CHANNEL_UNSPECIFIED = 0;
  CHANNEL_EMAIL = 1;
  CHANNEL_SLACK = 2;
  CHANNEL_SMS = 3;
}
```

전체 `.proto` 5개는 `msa-common-libs/events/src/main/proto/troica/{order,notification}/v1/` 참조.

#### 호환성 규칙

- 필드 추가 OK (proto3 default optional). 필드 번호는 **불변** — 변경 시 와이어 호환 깨짐
- enum 값 추가 OK. 미지원 값은 `*_UNSPECIFIED`로 fallback — 컨슈머가 처리
- breaking change → major bump (`v1` → `v2`). 양립 기간 동안 두 버전 package 공존

### 9.4 멱등성 처리

PostgreSQL 테이블:

```sql
CREATE TABLE notification_history (
  idempotency_key UUID PRIMARY KEY,
  channel VARCHAR(20) NOT NULL,
  template VARCHAR(64) NOT NULL,
  recipient JSONB NOT NULL,
  status VARCHAR(20) NOT NULL,     -- SENT / FAILED / SKIPPED
  attempt_count INT DEFAULT 0,
  last_error TEXT,
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

처리 로직 (의사 코드):

```java
@KafkaListener(topics = {"order.confirmed", "order.cancelled", "notification.requested"},
               groupId = "notification-service")
public void consume(ConsumerRecord<String, byte[]> record, Acknowledgment ack) {
    NotificationEvent event = deserialize(record);

    // 멱등성 체크
    if (history.existsByIdempotencyKey(event.getIdempotencyKey())) {
        log.info("Skip duplicate: {}", event.getIdempotencyKey());
        ack.acknowledge();
        return;
    }

    try {
        // Circuit Breaker로 감싸기 (Resilience4j)
        switch (event.getChannel()) {
            case EMAIL -> sesClient.send(event);
            case SLACK -> slackClient.send(event);
            case SMS -> smsClient.send(event);
        }
        history.recordSuccess(event);
    } catch (Exception e) {
        history.recordFailure(event, e);
        // retry 정책에 따라 ack 또는 nack
        throw e;
    }
    ack.acknowledge();
}
```

### 9.5 DLQ

Strimzi Kafka Connect의 `errors.deadletterqueue.topic.name` 또는 Spring Kafka의 `DeadLetterPublishingRecoverer` 사용. DLQ 토픽 `notification.dlq` 추가 정의.

---

## 10. AlertManager 라우팅 설정

`platform/30-kube-prometheus-stack/values.yaml` 발췌:

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: troica-slack-info
      group_by: [namespace, alertname, severity]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      routes:
        - receiver: pagerduty-critical
          matchers:
            - severity="critical"
          continue: true                      # critical은 Slack에도 미러링
        - receiver: troica-slack-alerts
          matchers:
            - severity="critical"
        - receiver: troica-slack-alerts
          matchers:
            - severity="warning"
        - receiver: troica-slack-info
          matchers:
            - severity="info"
    receivers:
      - name: pagerduty-critical
        pagerduty_configs:
          - service_key_file: /etc/alertmanager/secrets/pagerduty-key
            send_resolved: true
      - name: troica-slack-alerts
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack-webhook-alerts
            channel: '#troica-alerts'
            send_resolved: true
            title: '{{ template "slack.default.title" . }}'
            text: '{{ template "slack.default.text" . }}'
      - name: troica-slack-info
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack-webhook-info
            channel: '#troica-info'
            send_resolved: false

  alertmanagerSpec:
    secrets:
      - alertmanager-secrets             # ExternalSecrets로 prepare
```

### 10.1 Kafka consumer lag 알림 (계획서 E12)

`PrometheusRule`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kafka-consumer-lag
  namespace: monitoring
spec:
  groups:
    - name: kafka.lag
      rules:
        - alert: KafkaConsumerLagHigh
          expr: kafka_consumergroup_lag > 1000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Kafka consumer lag high: {{ $labels.consumergroup }} on {{ $labels.topic }}"
        - alert: KafkaConsumerLagCritical
          expr: kafka_consumergroup_lag > 10000
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Kafka consumer lag critical: {{ $labels.consumergroup }} on {{ $labels.topic }}"
```

> Strimzi의 Kafka exporter 또는 별도 `kafka-exporter` Helm chart 배포 필요.

---

## 11. Terraform 추가 사항 (`msa-provisioning`)

### 11.1 GitHub OIDC Provider + Role

```hcl
# oidc.tf
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "gha_ecr_push" {
  name = "troica-gha-ecr-push"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          "token.actions.githubusercontent.com:sub" =
            "repo:KTCloud-CloudNative-Troica-Team/msa-*:ref:refs/heads/main"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "gha_ecr_push" {
  role = aws_iam_role.gha_ecr_push.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["ecr:GetAuthorizationToken"]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:PutImage"
        ]
        Resource = "arn:aws:ecr:${var.region}:${var.account_id}:repository/msa/*"
      }
    ]
  })
}
```

### 11.2 ECR Repositories

```hcl
# ecr.tf
resource "aws_kms_key" "ecr" {
  description             = "Troica ECR encryption key"
  enable_key_rotation     = true
  deletion_window_in_days = 7
}

locals {
  services = ["user", "product", "inventory", "order", "notification", "api-gateway"]
}

resource "aws_ecr_repository" "service" {
  for_each             = toset(local.services)
  name                 = "msa/${each.key}-service"
  image_tag_mutability = "IMMUTABLE"

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  image_scanning_configuration {
    scan_on_push = true
  }
}

resource "aws_ecr_lifecycle_policy" "service" {
  for_each   = aws_ecr_repository.service
  repository = each.value.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 30 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 30
      }
      action = { type = "expire" }
    }]
  })
}
```

### 11.3 VPC Endpoint for ECR (계획서 5.1 일관성)

```hcl
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"                  # ECR이 S3에 의존
  route_table_ids   = aws_route_table.private[*].id
}
```

> ECR은 내부적으로 S3에 이미지 레이어를 저장하므로 S3 Gateway Endpoint도 함께 필요.

### 11.4 Ansible: ECR credential provider (kubelet)

`ansible/roles/k8s/templates/credential-provider-config.yaml.j2`:

```yaml
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
providers:
  - name: ecr-credential-provider
    matchImages:
      - "{{ aws_account_id }}.dkr.ecr.{{ aws_region }}.amazonaws.com"
    defaultCacheDuration: "12h"
    apiVersion: credentialprovider.kubelet.k8s.io/v1
```

kubelet 시작 옵션에 `--image-credential-provider-config=/etc/kubernetes/credential-provider-config.yaml`와 `--image-credential-provider-bin-dir=/usr/local/bin` 추가. `ecr-credential-provider` 바이너리는 [cloud-provider-aws](https://github.com/kubernetes/cloud-provider-aws) 릴리스에서 다운로드.

EC2 instance profile에 `AmazonEC2ContainerRegistryReadOnly` managed policy 부여 필요.

---

## 12. msa-common-libs 배포·소비 흐름

### 12.0 멀티모듈 구조 (Phase 2에서 확정)

| 서브모듈 | groupId:artifactId | 내용 |
|---------|---------------------|------|
| `common`       | `com.troica.msa:common`       | JPA/QueryDSL config, Base 엔티티, 공통 예외/시간/Backoff 유틸 |
| `client-redis` | `com.troica.msa:client-redis` | Redisson 분산락 + `@IdempotentEvent` aspect |
| `client-ses`   | `com.troica.msa:client-ses`   | AWS SES adapter + SendMailUseCase |
| `events`       | `com.troica.msa:events`       | Kafka 5개 토픽 Protobuf 스키마 + codegen된 Java/Kotlin 클래스 |

각 서비스는 **필요한 모듈만** 의존 → 불필요한 ClassPath 부피 회피.

### 12.0.1 빌드 환경 (Phase 2 검증된 조합)

- **Kotlin 2.1.0 + Gradle 8.10.2 + JVM 21** — 모노레포의 Kotlin 2.3.20 + Gradle 9.0.0 조합은 `BuildUtilKt.clearJarCaches()` `NoClassDefFoundError` (Kotlin Build Tools API ABI 충돌)로 빌드 실패. 안정 조합으로 내려 진행. 소비자(Phase 3+ 서비스 레포)는 동등 또는 상위 Kotlin/Gradle 사용 가능 — Kotlin metadata는 호환.
- **Spring Boot BOM 3.5.13** 을 `io.spring.dependency-management` 플러그인으로 import (의존성 버전 통일). 라이브러리이므로 `org.springframework.boot` 플러그인은 **적용하지 않음** (bootJar 불필요). 3.5.x OSS support는 2026-06-30 EOL — Phase 6 이후 SB 3.6/4.0 마이그레이션 트랙 별도 검토.

### 12.1 publish (msa-common-libs 측)

루트 `build.gradle.kts` 핵심 — `subprojects` 블록에서 모든 모듈에 `maven-publish` 일괄 적용:

```kotlin
subprojects {
    apply(plugin = "org.jetbrains.kotlin.jvm")
    apply(plugin = "io.spring.dependency-management")
    apply(plugin = "maven-publish")
    apply(plugin = "java-library")

    the<io.spring.gradle.dependencymanagement.dsl.DependencyManagementExtension>().imports {
        mavenBom("org.springframework.boot:spring-boot-dependencies:3.5.13")
    }

    extensions.configure<JavaPluginExtension> {
        toolchain { languageVersion.set(JavaLanguageVersion.of(21)) }
        withSourcesJar()
        withJavadocJar()
    }

    extensions.configure<PublishingExtension> {
        publications {
            register<MavenPublication>("library") {
                from(components["java"])
                artifactId = project.name        // common / client-redis / client-ses / events
            }
        }
        repositories {
            maven {
                name = "GitHubPackages"
                url = uri("https://maven.pkg.github.com/KTCloud-CloudNative-Troica-Team/msa-common-libs")
                credentials {
                    username = System.getenv("GITHUB_ACTOR")
                        ?: providers.gradleProperty("gpr.user").orNull
                    password = System.getenv("GITHUB_TOKEN")
                        ?: providers.gradleProperty("gpr.token").orNull
                }
            }
        }
    }
}
```

`.github/workflows/publish.yml`:

```yaml
name: Publish

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: gradle }
      - uses: gradle/actions/setup-gradle@v4
      - name: Extract version from tag
        id: ver
        run: |
          TAG="${GITHUB_REF_NAME}"
          echo "version=${TAG#v}" >> "$GITHUB_OUTPUT"
      - name: Build all modules
        run: ./gradlew build -Pversion=${{ steps.ver.outputs.version }} --no-daemon
      - name: Publish to GitHub Packages
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: ./gradlew publish -Pversion=${{ steps.ver.outputs.version }} --no-daemon
```

### 12.2 consume (각 서비스 레포 측)

`build.gradle.kts`:

```kotlin
repositories {
    mavenCentral()
    maven {
        url = uri("https://maven.pkg.github.com/KTCloud-CloudNative-Troica-Team/msa-common-libs")
        credentials {
            username = System.getenv("GITHUB_ACTOR") ?: project.findProperty("gpr.user")?.toString()
            password = System.getenv("GITHUB_TOKEN") ?: project.findProperty("gpr.token")?.toString()
        }
    }
}

dependencies {
    // 필요한 모듈만 선택적으로 의존
    implementation("com.troica.msa:common:0.1.0")
    implementation("com.troica.msa:events:0.1.0")          // Kafka producer/consumer 필요한 서비스
    implementation("com.troica.msa:client-redis:0.1.0")    // 분산락/멱등 필요한 서비스
    implementation("com.troica.msa:client-ses:0.1.0")      // 이메일 발송 필요한 서비스 (notification-service)
}
```

로컬 개발 시에는 `~/.gradle/gradle.properties`에 `gpr.user` / `gpr.token`(packages:read 권한 PAT) 등록.

---

## 13. 단계별 작업 순서 (Claude Code 진행 가이드)

진행 시 **각 단계가 끝나면 검증 체크포인트를 거치고 다음 단계로 진행**한다. 한 번에 다 만들지 말 것.

### Phase 0: Terraform 인프라 정정 (우문주 영역, 사전 작업)

1. `msa-provisioning`에 §11.1, §11.2, §11.3 추가
2. `terraform apply`로 OIDC Provider + 6개 ECR repo + VPC Endpoint 생성
3. Ansible playbook의 `k8s` role에 §11.4 ECR credential provider 추가
4. GitHub Org Secrets 등록: `AWS_ACCOUNT_ID`, `MANIFEST_PAT`

### Phase 1: 매니페스트 레포 골격 (PoC 전 사전 작업)

1. `msa-argocd-manifest`에 §2 디렉토리 구조 빈 껍데기 생성
2. `applications/charts/microservice/` Helm 차트 작성 (§3)
3. `applications/appset.yaml` 작성 (§5) — **lint만 하고 apply 보류**
4. `projects/market-project.yaml`, `projects/platform-project.yaml` 작성

### Phase 2: msa-common-libs 분리

1. 새 레포 `msa-common-libs` 생성
2. **멀티모듈 Gradle** (4 서브모듈: `common`, `client-redis`, `client-ses`, `events`) — §12.0
3. 기존 모노레포에서 `common/`, `client-redis/`, `client-ses/` 소스 이전
4. `events/` 모듈에 5개 `.proto` 스키마 작성 — §9.3 매핑 표 참조
5. `com.google.protobuf` Gradle 플러그인으로 codegen (Java + Kotlin)
6. §12.1 publish workflow 작성
7. **빌드 환경**: Kotlin 2.1.0 + Gradle 8.10.2 + JVM 21 — §12.0.1 참조 (모노레포 버전 그대로면 빌드 실패)
8. `./gradlew build publishToMavenLocal` 로 로컬 검증
9. `v0.1.0` 태그 push → GitHub Packages에 published 확인

### Phase 3: 첫 서비스 PoC — order-service

가장 복잡한 서비스부터. 여기서 막히는 부분이 다른 서비스에서도 막힌다.

1. 새 레포 `msa-order-service` 생성
2. 기존 모노레포에서 `order-service`, `order` 도메인 모듈 이전 (`inventory-event`는 inventory 도메인 — Phase 4에서 inventory-service로)
3. **멀티모듈 Gradle 구조 유지** (`order/` 도메인 라이브러리 + `order-service/` Spring Boot 앱) — 모노레포의 모듈 경계 그대로 사용
4. `build.gradle.kts`에 §12.2 GitHub Packages repository 추가, `:common`/`:client-redis`/`:events` 등 monorepo 모듈 의존을 `com.troica.msa:*` 의존으로 치환
5. **worker 분기 신규 구현** (BACKLOG R-07): `@Profile("worker") @Scheduled` 빈 1개 + `application-worker.yaml`(web off, gRPC port=-1) + `@EnableScheduling` 적용. 기존 `processAll()`은 그대로 재사용
6. `application-{dev,prod,worker}.yaml` 분리, base는 환경변수 driven (`ORDER_DB_HOST` 등)
7. `Dockerfile` 작성 (§8) — JarLauncher 경로는 `org.springframework.boot.loader.launch.JarLauncher` (SB 3.5.13 검증됨, R-06 해결)
8. `.github/workflows/ci.yml` 작성 (§7) — push-gated jobs는 Phase 0 완료까지 빨갛게 떨어짐 (의도된 동작)
9. **gradlew를 git index에 mode 100755로 커밋** (BACKLOG의 Phase 2 교훈)
10. 로컬 검증: `./gradlew build -x test` SUCCESS, bootJar MANIFEST의 Main-Class 확인
11. PR 1개 만들어서 CI 검증 — common-libs가 GitHub Packages에 publish되어 있어야 함 (`v0.1.0` 태그 → publish workflow)
12. main 머지 → push trigger 동작 (Phase 0 완료 전에는 build-test까지만 성공)
13. 매니페스트 레포에 `applications/values/order-service/{values, values-dev, values-prod}.yaml` 추가 (§4) — 별도 PR
14. ApplicationSet 자동으로 `order-service-dev` / `order-service-prod` Application 생성
15. (Phase 0 후) main 머지 시 CI가 매니페스트 레포 `values-dev.yaml` 자동 commit, `values-prod.yaml`은 PR 생성
16. ArgoCD UI에서 `order-service-dev` Application Healthy + Synced
17. K8s에 order-service Pod 2종 (`order-service` api + `order-service-worker`) 떠야 함

### Phase 4: 나머지 서비스 복제

Phase 3에서 검증된 패턴으로 나머지 5개 서비스 동일하게:
- `msa-user-service`
- `msa-product-service`
- `msa-inventory-service`
- `msa-notification-service` (§9 추가 구현)
- `msa-api-gateway`

각 서비스마다 (1) 코드 이전, (2) Dockerfile, (3) CI workflow, (4) values 추가, (5) 배포 검증.

### Phase 5: Kafka 토픽 + AlertManager + 관측성

1. `platform/50-kafka-cluster/topics/` 5개 KafkaTopic CRD 작성 (§9.2)
2. `platform/30-kube-prometheus-stack/values.yaml`에 §10 AlertManager 라우팅 추가
3. `PrometheusRule`로 Kafka consumer lag 알림 추가 (§10.1)
4. Slack Webhook URL, PagerDuty service key는 ExternalSecrets로 주입

### Phase 6: 마이그레이션 정리

1. 기존 `msa-spring-boot` 모노레포 archive (read-only)
2. README에 마이그레이션 내역 명시
3. 발표 자료 업데이트 (polyrepo 다이어그램)

---

## 14. 검증 체크리스트 (각 Phase 끝나면 확인)

### Phase 0 끝
- [ ] `aws iam get-role --role-name troica-gha-ecr-push` 응답 있음
- [ ] `aws ecr describe-repositories` 6개 출력
- [ ] EC2 인스턴스에서 `aws ecr get-login-password` 성공
- [ ] kubelet credential provider 설정으로 ECR private pull 성공

### Phase 1 끝
- [ ] `helm template applications/charts/microservice -f applications/values/order-service/values.yaml` 에러 없이 렌더링
- [ ] `kubectl --dry-run=client apply -f applications/appset.yaml` 통과

### Phase 3 끝 (PoC 검증)
- [ ] order-service main 머지 시 ECR에 `main-<sha>` 이미지 push
- [ ] Trivy CRITICAL/HIGH 0건
- [ ] manifest 레포 `values-dev.yaml`에 tag 자동 commit
- [ ] manifest 레포에 prod PR 생성 (머지 안 함)
- [ ] ArgoCD UI에서 `order-service-dev` Healthy + Synced
- [ ] `kubectl get deploy -n market-dev` → `order-service`, `order-service-worker` 둘 다 Running
- [ ] order-worker가 Outbox 폴링하고 `order.pending` 토픽에 publish 확인

### Phase 5 끝
- [ ] `kubectl get kafkatopic -n kafka` 5개 토픽 Ready
- [ ] AlertManager UI에서 receiver 3개 (`pagerduty-critical`, `troica-slack-alerts`, `troica-slack-info`) 활성
- [ ] 테스트 알림 fire → Slack에 들어옴, critical 시 PagerDuty도 동시 트리거

---

## 15. 알려진 위험 & 대응

| 위험 | 영향 | 대응 |
|------|------|------|
| GitHub Packages rate limit (시간당 1000 unauthenticated, authenticated는 더 높음) | CI에서 common-libs pull 실패 | 모든 CI에서 `GITHUB_TOKEN` 인증, 의존성 캐싱 |
| ECR IMMUTABLE tag 정책 + 동일 sha 재push | push 실패 | sha 7자리 + 브랜치 prefix(`main-`)로 충돌 방지, 충돌 시 새 commit로 재시도 |
| ArgoCD multi-source 미지원 버전 사용 시 | platform Application 일부 동작 안 함 | ArgoCD v2.6+ 확정, `kubectl -n argocd get deploy argocd-server -o yaml | grep image:`로 버전 확인 |
| `values-dev.yaml`과 `values-prod.yaml`이 동시에 머지될 때 race | PR과 push 충돌 | 두 job이 같은 main을 건드리지만 다른 파일이라 충돌 없음. 단, 빠른 연속 push 시 `git pull --rebase` 필요 |
| kubelet ECR token 만료(12h) | image pull 실패 | credential provider의 `defaultCacheDuration: "12h"` 설정, instance profile 권한 확인 |
| Spring Boot 3.5 JarLauncher 경로 | 컨테이너 시작 실패 | Dockerfile §8 주석 참고, 실제 jar에서 manifest 확인 |
| common-libs 버전 분기 | 5개 서비스 의존성 buf지옥 | SemVer + breaking change 시 메이저 bump, 같은 sprint에 모든 서비스 일괄 업데이트 |

---

## 16. 빠른 참조 — 자주 쓰는 명령어

```bash
# ECR 로그인 (로컬 디버깅)
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin \
    <ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com

# Helm 차트 로컬 렌더링 검증
helm template applications/charts/microservice \
  -f applications/values/order-service/values.yaml \
  -f applications/values/order-service/values-dev.yaml

# ApplicationSet 미리보기 (dry-run)
argocd appset get market-services --output yaml

# Kafka 토픽 확인
kubectl exec -it -n kafka market-kafka-kafka-0 -- \
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# AlertManager receiver 테스트
kubectl exec -it -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager-0 -- \
  amtool alert add alertname=TestAlert severity=warning
```

---

## 부록 A: 이번 결정에서 의도적으로 채택 안 한 것 (참고용)

- **ArgoCD Image Updater**: CI push 방식이 더 명시적이고 디버깅 쉽다. Image Updater는 학습 비용 + 추가 컴포넌트 부담.
- **GHCR**: AWS 인프라 + KMS + VPC Endpoint 일관성 위해 ECR로. 검증된 결정.
- **Grafana Alerting**: 계획서가 AlertManager 명시. 둘 다 가능하지만 일관성 우선.
- **마이크로서비스별 Helm 차트 분리**: 동질적이라는 계획서 판단에 따라 공통 차트 1개로. 차후 분기 필요해지면 그때 split.
- **payment-service / shipping-service**: 4주 일정 + 계획서 의도. order-service 내 상태 전이로 시뮬레이션.

---

**문서 버전**: 1.0
**작성일**: 2026-05-12
**대상 도구**: Claude Code (Anthropic CLI)
