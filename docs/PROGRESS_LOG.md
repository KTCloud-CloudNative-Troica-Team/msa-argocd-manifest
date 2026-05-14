# Troica polyrepo migration — Progress Log

> 본 문서는 컨텍스트 압축 시점에 스냅샷으로 작성되었습니다 (2026-05-12).
> 자세한 task/R 항목은 `BACKLOG.md` 참고. SPEC은 `TROICA_SPEC.md`.

## 0. 빠른 컨텍스트 회복 (Quick Recovery)

새 세션 시작 시 다음 3개를 순서대로 읽으면 95% 컨텍스트 복원:

1. 이 문서 (`PROGRESS_LOG.md`) — Phase 진행 상황 + 결정 누적
2. `BACKLOG.md` — R 항목 (해결/미해결) + Phase별 작업 분해
3. `TROICA_SPEC.md` — 단일 진실 소스 (아키텍처/계약/규약)

추가 메모리 (Claude 사용자 자동 메모리):

- `~/.claude/projects/.../memory/project_troica_msa.md` — 프로젝트 컨텍스트
- `~/.claude/projects/.../memory/project_troica_progress.md` — 본 문서의 인덱스
- `~/.claude/projects/.../memory/feedback_no_push_monorepo.md` — msa-spring-boot:main read-only
- `~/.claude/projects/.../memory/feedback_gradlew_executable.md` — gradlew mode 100755 git index 필수
- `~/.claude/projects/.../memory/project_istio_over_traefik.md` — Phase 5에서 Istio 교체
- `~/.claude/projects/.../memory/reference_troica_spec.md` — SPEC 변경 시 BACKLOG 동시 갱신

## 1. Phase 진행 상태 (2026-05-12 기준)

| Phase | 내용 | 상태 |
| --- | --- | --- |
| Phase 1 | ArgoCD 매니페스트 골격 (chart, appset, project) | ✅ 머지 |
| Phase 2 | common-libs 멀티모듈 + Protobuf 빌드 + GH Packages publish | ✅ 머지 |
| Phase 3 | order-service + worker (Outbox + Kafka publisher) | ✅ 머지 |
| Phase 4 | 6개 서비스 polyrepo 분리 — product/order/inventory/user | ✅ 머지 |
| Phase 4 | auth-service / api-gateway PR | ⏳ **미머지** (사용자 액션 대기) |
| Phase 0 | AWS OIDC + ECR + VPC Endpoint | 🔄 PR 1 머지, PR 2 (ansible) 미머지, **PR 3 (S3 backend + prevent_destroy) 진행 중** |
| Phase 5 | Kafka topics CRD + AlertManager + Istio platform | ⏸ 미시작 |
| Phase 6 | 모노레포 archive + 발표 자료 | ⏸ 미시작 |

## 2. 현재 진행 중 — PR 3: phase-0/permanent-resources-and-backend

**Repo**: `msa-provisioning`

**왜 필요한가**: 예산 제약 (1인당 66,666원, 사용자 이미 9,830원 사용 → 남은 51,000원) 때문에
`terraform apply → 검증 → destroy` 사이클을 반복해야 함.
그런데 OIDC role ARN / ECR repo URL을 GitHub Org Secrets에 등록해야 하므로
destroy/apply 사이클마다 ARN이 바뀌면 안 됨.

**해결**: 자원 분류 + prevent_destroy + S3 backend

### 영구 자원 (월 ~$1, lifecycle prevent_destroy)

- IAM OIDC Provider (`aws_iam_openid_connect_provider.github`)
- IAM Role `troica-gha-ecr-push`
- ECR 6 repos (user-service, auth-service, product-service, inventory-service, order-service, api-gateway)
- ECR KMS key

**ARN/URL 안정성 검증**: 모두 name-based 자원. destroy → apply 후에도 ARN/URL 동일.
→ GitHub Secrets 영구 유효.

### 임시 자원 (월 ~$320 24/7, `scripts/destroy-temp.sh -target`)

- EC2 instances (master/worker/bastion × 7)
- NAT Gateway
- NLB
- EBS volumes
- EIP
- VPC Endpoint × 2 (ecr.api, ecr.dkr — 시간당 fixed cost)

### 수동 자원 (Terraform 외부)

- S3 backend bucket — 1회 생성, terraform 외부 (닭과 달걀 회피)

### PR 3 구성 (5개 변경)

1. `terraform/backend.tf` 신규 — S3 backend, `use_lockfile = true` (Terraform 1.10+ native S3 lock, DynamoDB 불필요)
2. `terraform/oidc.tf` — OIDC provider + IAM role에 `lifecycle { prevent_destroy = true }`
3. `terraform/ecr.tf` — KMS key + ECR repos에 `prevent_destroy = true`, ECR에 `force_delete = true` 추가
4. `scripts/destroy-temp.sh` 신규 — `terraform destroy -target=...` 으로 임시 자원만 정리
5. `README.md` — bootstrap S3 절차 + 사이클 워크플로우 문서화

S3 bucket suffix는 사용자 결정 대기 → placeholder `<SUFFIX>` 사용.

## 3. 결정 누적 (Single Source of Truth)

### 기술 스택

- **Spring Boot 3.5.13** (기존 3.3.0 → CVE-2025-22235 HIGH 7.3 fix). OSS EOL 2026-06-30 (R-18).
- **Kotlin 2.1.0** + **Gradle 8.10.2** + JVM 21.
  - Kotlin 2.3.x + Gradle 9.x = `BuildUtilKt.clearJarCaches` NoClassDefFoundError (JetBrains 버그). 3가지 workaround 실패. **2.1.0 pin** (R-09/R-16).
- **Spring Cloud 2025.0.2** (Northfields, 2026-04-02 릴리스) — SB 3.5.x 공식 페어. api-gateway 한정.
  - 2025.1.0은 Boot 4.x 전용.
- **gRPC** — auth-service의 `CheckValidity`, api-gateway의 BFF.
- **Kafka wire = JSON** (Q4 a). 토픽명 = SPEC 표준 (Q5 a):
  `order.pending`, `order.inventory-reserved`, `order.confirmed`, `order.cancelled`.

### 라이브러리

- **common-libs v0.3.0** — 2 모듈 (`common` + `events`).
- **client-redis** → JitPack `com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2` (D2).
  - Kotlin 2.3.x 메타데이터 → consumer는 `-Xskip-metadata-version-check` 필요.
- **client-ses 제거** (D3).
- **GH Packages** publish — user-service는 `:user` 모듈만 publish.

### 서비스 구성

- **auth-service 신규 레포** (D1). `user/auth/identification` 중 identification 폐기.
- **api-gateway = BFF + SC Gateway 혼합** (Q1 c).
  - BFF: REST → gRPC (auth, user, product, order, inventory)
  - SC Gateway routes: 일부 path는 백엔드로 직접 프록시 (e.g. `/admin/v1/orders/**`)
  - proto 자체 보유.
- **notification-service 폐기** (D6 archive). Grafana → Slack 알림 (Q3).
- **order**: state machine 7-state (Q6) + OrderStatusOutbox + admin endpoint.
  - PENDING → INVENTORY_RESERVED → PAID → SHIPPED → CONFIRMED / FAILED / CANCELLED
- **inventory**: Event Sourcing + Spring Batch worker (Q7).
  - `@Profile("worker")` Batch Job + chunk-oriented ItemReader/Processor/Writer.

### CI/CD

- **Trivy action v0.36.0** pinned (R-14: v0.0.1~v0.34.2 supply chain attack).
- **gradlew mode 100755** in git index (Windows에서 만들면 100644, Linux CI에서 Permission denied).
- **AWS_DEPLOYMENTS_ENABLED Org Variable gate** (R-19) — 모든 7개 push-gated workflow에:
  `&& vars.AWS_DEPLOYMENTS_ENABLED == 'true'`.

### Infra

- **AWS OIDC** — GitHub Actions → AWS, no long-lived access key.
  - Trust policy: `repo:KTCloud-CloudNative-Troica-Team/msa-*:ref:refs/heads/main`
- **VPC Endpoint** — kubelet (private subnet) → ECR private repo 이미지 pull.
  - `ecr.api` + `ecr.dkr` Interface endpoint × 2 (~$14/월)
  - S3 Gateway endpoint (무료, ECR layer storage)
- **Terraform backend** — S3 + native lockfile (Terraform 1.10+, `use_lockfile=true`).
  - 3인 협업 + 비용 제약 + DynamoDB 불필요.

## 4. 활성 R 항목 (요약 — 자세한 건 BACKLOG)

### 해결됨

R-01, R-02, R-06, R-07, R-10, R-11, R-14, R-17.

### 미해결

| 항목 | 내용 | 상태 |
| --- | --- | --- |
| R-04 | user-api-gateway 명명 | 관리적, 무영향 |
| R-05 | notification 그린필드 | D6 archive로 해결 |
| R-08 | 모노레포 archive | Phase 6 |
| R-09 | Kotlin 2.3.x bug | 2.1.0 pin으로 회피, JetBrains 수정 대기 |
| R-12 | docker build 로컬 미검증 | CI green 확인됨 |
| R-13 | Kafka JSON → Protobuf 마이그레이션 | Phase 4 후 연기 |
| R-15 | GitHub Actions SHA pin + Dependabot | 점진 |
| R-16 | Kotlin 2.3.x build | R-09 동의 |
| R-18 | SB 3.5 OSS EOL 2026-06-30 | 모니터링 |
| R-19 | AWS_DEPLOYMENTS_ENABLED 활성화 | Phase 0 머지 + Org Variable 등록 시 자동 해결 |
| R-20 | JitPack client-redis 모니터링 | 1.x 유지 |
| R-21~25 | Phase 4 Q6/Q7 작업 | 대부분 해결 |
| R-26 | manifest image.repository placeholder ACCOUNT_ID 일괄 치환 | Phase 0 적용 후 |

## 5. 사용자 액션 대기 사항

1. **msa-auth-service** `phase-4/initial-setup` PR 머지
2. **msa-api-gateway** `phase-4/initial-setup` PR 머지
3. **msa-argocd-manifest** auth-service-values + api-gateway-values + phase-0/docs 머지 (대부분 완료)
4. **Phase 0 적용 절차**:
   - a. (사용자) S3 bucket 수동 생성 (suffix 결정)
   - b. (사용자) PR 3 (S3 backend + prevent_destroy) 머지
   - c. (사용자) `terraform init -migrate-state` (로컬 → S3)
   - d. (사용자) `terraform apply` (영구 + 임시 자원 모두)
   - e. (사용자) GitHub Org Secrets/Variable 등록:
     - `AWS_ACCOUNT_ID` (Secret)
     - `MANIFEST_PAT` (Secret, ArgoCD manifest 푸시용)
     - `AWS_DEPLOYMENTS_ENABLED=true` (Variable)
   - f. (사용자) 검증 후 `scripts/destroy-temp.sh` 실행으로 임시 자원만 정리 (영구 자원 유지)
   - g. (다음 작업) 재개 시 `terraform apply` 한 번 더 → 임시 자원만 재생성

## 6. 핵심 에러 + 해결 내역 (재발 방지용)

- **gradlew 100644 → Linux CI Permission denied**
  → `git update-index --chmod=+x gradlew`. 모든 서비스 레포 초기 commit에 반영 (메모리 등록).
- **Trivy @0.24.0 미존재 + 공급망 공격**
  → `@v0.36.0` (post-incident, v-prefix).
- **Kotlin 2.3.x + Gradle 9.x BuildUtilKt 에러**
  → 3 workaround (runViaBuildToolsApi=false / useClasspathSnapshot=false / incremental=false) 모두 실패.
  → Kotlin 2.1.0 + Gradle 8.10.2 pin.
- **JitPack client-redis (Kotlin 2.3.x metadata) + 컴파일러 (Kotlin 2.1.0) 호환**
  → `freeCompilerArgs.addAll("-Xskip-metadata-version-check")`.
- **common-libs `0.x.0-SNAPSHOT` GH Packages 부재**
  → 머지 직전 stable `0.x.0`으로 bump (반복 패턴).
- **CI 영구 red — OIDC AssumeRole role 없음**
  → 모든 push-gated 조건에 `&& vars.AWS_DEPLOYMENTS_ENABLED == 'true'` 추가 (R-19).
- **terraform fmt 오류 — JSON StringLike 줄 분리**
  → 한 줄로 합쳐서 해결.
- **PR 2가 PR 1 자원 참조 → terraform validate 실패**
  → PR 2를 PR 1 브랜치에 rebase (stacked PR).
- **Spring Batch ItemWriter Kotlin lambda 타입 추론 실패**
  → `object : ItemWriter<...> { override fun write(...) }` 명시.
- **inventory-service가 :inventory-event 모듈 transitive 미인식**
  → `implementation(project(":inventory-event"))` 명시 추가.

## 7. Next 진행 (Phase 0 적용 완료 후)

- **Phase 5**:
  - Kafka topics CRD (Strimzi KafkaTopic) for 4 topics
  - AlertManager + Slack receiver
  - 관측성 (Prometheus + Grafana + Loki)
  - Istio platform 배포 + Traefik 제거 (R-Istio 메모리)
- **Phase 6**:
  - 모노레포 (`msa-spring-boot`) archive
  - 발표 자료 (architecture diagram + decision log + 비용 분석)

---

*Last updated: 2026-05-12, by Troica polyrepo migration session.*

---

## 2026-05-14 스냅샷 — Phase 5 진행 (R-35 매니페스트 + 평가요소 cover)

### 머지 완료 (main)
- R-43 PDB / R-44 ADR-0009 / R-46 Trivy config / R-48 RBAC / R-49 NetworkPolicy / R-50 Rate Limit / R-41 Fallback / R-45 SonarCloud (sonarcloud.io 셋업 + ADR-0011 wait=false) / R-57 단위 테스트 (7 polyrepo ~48 케이스) / ADR-0010 보안 알림 / ADR-0011 SonarCloud 정책

### Open PR (미머지)
- `feat/r-35-platform-manifests` — **R-35 매니페스트 22 파일 일괄 작성** (cert-manager → external-secrets-config). cluster up 전 사전 작성. 머지 후 cluster up + ArgoCD sync 로 자동 reconcile.

### 평가요소 cover 현황 (필수 18)
- ✅ 13 / 🔄 3 (R-44 검증, R-47 AlertManager, R-49 검증) / ⏸ 2 (R-42, R-41 cluster 검증)
- cluster up + Slack 채널 생성 시 18/18 도달

### 다음 작업 — R-42 (사용자 직접 지시, 2026-05-14)
- **R-42 Postman Collection + Newman CI + Grafana 대시보드** (cluster up 전 가능)
- 평가 cover: 기본 (3)-2 (필수) + (3)-3 (필수) + (3)-5 (선택)
- 작업 원칙: **추측 X, 실 코드 확인 기반**
  - api-gateway 실 endpoint = `src/main/kotlin/.../adapter/presentation/web/inbound/*RestController.kt` 직접 확인
  - SecurityConfig 확인
  - Spring Boot Actuator metric 이름 확인 (Micrometer)

### 사용자 사전 작업 (cluster up 직전 필요)
- AWS Secrets Manager 9 secrets 등록 (troica/db/* × 6 + troica/auth/jwt-secret + troica/slack/webhooks + troica/grafana/admin). 30일 free trial 활용 권장
- Slack `#security-report` 채널 생성 + Incoming Webhook 발급
- msa-provisioning terraform 에 EC2 node IAM role 의 `SecretsManager:GetSecretValue` 권한 추가 (R-25 운영)

### Phase 5 / Phase 6 구분 (BACKLOG 재구성 후)
- **Phase 5** = 평가 필수 100% cover (cluster up + 검증)
- **Phase 6** = 심화 선택 (R-51 KEDA / R-52 Rollouts / R-53 OPA / R-54 ZAP / R-55 Falco / R-56 Incident Response / R-58 Testcontainers) + 운영 전환 (R-08 archive / R-15 SHA pin / R-18 SB EOL / R-40 frontend 호스팅) + P6.1 발표 자료

### 보류 — Phase 6 task 옵션
- 발표 임팩트 ROI: R-52 (Rollouts, ArgoCD 이미 있음) → R-51 (KEDA) → R-53 (OPA) → 나머지

*Last updated: 2026-05-14 PM, R-35 매니페스트 PR 푸시 시점.*
