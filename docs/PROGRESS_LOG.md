# Troica polyrepo migration — Progress Log

> 본 문서는 컨텍스트 압축 시점에 스냅샷으로 작성됩니다.
> 자세한 task/R 항목은 [BACKLOG.md](./BACKLOG.md) 참고. SPEC은 [TROICA_SPEC.md](./TROICA_SPEC.md).

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
- `~/.claude/projects/.../memory/project_istio_over_traefik.md` — Phase 5에서 Istio 교체 (완료)
- `~/.claude/projects/.../memory/reference_troica_spec.md` — SPEC 변경 시 BACKLOG 동시 갱신

---

## 1. Phase 진행 상태 (2026-05-18 기준 — 평가 18/18 완성)

| Phase | 내용 | 상태 |
|---|---|---|
| Phase 0 | AWS Phase 0 영구 + 임시 자원 (Terraform) + ECR + OIDC + VPC Endpoint + S3 backend | ✅ 머지 |
| Phase 1 | ArgoCD 매니페스트 골격 (chart, appset, project, bootstrap) | ✅ 머지 |
| Phase 2 | common-libs 멀티모듈 (`common` + `events`) + Protobuf + GH Packages publish | ✅ 머지 |
| Phase 3 | order-service polyrepo + worker (Outbox + Kafka publisher) | ✅ 머지 |
| Phase 4 | 6 service polyrepo + msa-frontend 신규 + auth-service / api-gateway 신규 | ✅ 머지 |
| Phase 5 | platform 19 Application + 평가 18/18 cover (RBAC / NetworkPolicy / Trivy / SonarCloud / Slack / PDB / E2E / Observability / Service Mesh) | ✅ **완료 (2026-05-18)** |
| Phase 6 | 선택 평가요소 (KEDA / Canary / OPA / ZAP / Falco / IR / Testcontainers / R-50 anonymous fallback 정정) + 운영 전환 (frontend hosting / monorepo archive / SHA pin / SB EOL) | 📌 대기 |

## 2. 평가 18 통과 결과 (2026-05-18)

**필수 18 / 18 ✅**:
- 기본 필수 9 / 9 ✅ — Domain (1) / Communication (2) / Test+Observability (3)
- 심화 필수 9 / 9 ✅ — Operations (1) / Security (2) / IAM (3)

**선택 보너스 4 ✅**:
- 기본 (1)-5 Event Sourcing (R-22 inventory)
- 기본 (2)-4 Resilience4j Circuit Breaker (R-41)
- 기본 (3)-5 Newman CLI in CI (R-42)
- 기본 (3)-6 Kafka consumer lag (R-35)

**Phase 6 으로 이동 (선택, 평가 영향 X)**:
- 기본 (2)-5 Rate Limit (R-50 (B)) — header 노출 확인 단 burst 시연은 anonymous fallback key 정정 후

## 3. 결정 누적 (Single Source of Truth)

### 기술 스택

- **Spring Boot 3.5.13** (3.3.0 → CVE-2025-22235 HIGH 7.3 fix). OSS EOL 2026-06-30 (R-18).
- **Kotlin 2.1.0** + **Gradle 8.10.2** + JVM 21.
  - Kotlin 2.3.x + Gradle 9.x = `BuildUtilKt.clearJarCaches` NoClassDefFoundError (JetBrains 버그). 3 workaround 실패. **2.1.0 pin** (R-09/R-16).
- **Spring Cloud 2025.0.2** (Northfields) — SB 3.5.x 공식 페어. api-gateway 한정.
- **gRPC** — auth-service `CheckValidity` + api-gateway BFF.
- **Kafka wire = JSON** (Q4 a, ADR-0003). 토픽명 = SPEC 표준 (Q5 a, ADR-0004): `order.pending`, `order.inventory-reserved`, `order.confirmed`, `order.cancelled`.

### 라이브러리

- **common-libs v0.4.0** — `common` (servlet 제거) + `events`. api-gateway 측 WebFlux 호환.
  - 5 service (auth/user/product/inventory/order) 측은 v0.3.1 servlet 호환 그대로 유지.
- **client-redis** → JitPack `com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2` (D2, ADR-0002).
- **client-ses 제거** (D3, ADR-0002).
- **micrometer-registry-prometheus** — 6 polyrepo build.gradle.kts 측 추가 (R-65). `/actuator/prometheus` endpoint 활성화 → ServiceMonitor scrape → Grafana Troica dashboard panel data.

### 서비스 구성

- **auth-service 신규 레포** (D1, ADR-0001). JWT 발급 + `CheckValidity` gRPC.
- **api-gateway = BFF + SC Gateway 혼합** (Q1 c, ADR-0005).
  - BFF (gRPC client): auth (`/api/v1/auth/{signup,signin,check}`), product, order, inventory
  - SC Gateway routes (REST reverse proxy): user (`/api/v1/users/**`), order admin (`/admin/v1/orders/**`)
  - R-50 default-filters: `/api/v1/users` + `/admin/v1/orders` 측 RequestRateLimiter (burst 10/3 차등, replenish 1/s)
  - R-41 Circuit Breaker: `InventoryQueryService` `runCatching` fallback
- **notification-service 폐기** (D6 archive, ADR-0001). Grafana → Slack 알림 (Q3) + AlertManager severity routing (ADR-0010).
- **order**: state machine 7-state (Q6, ADR-0006) + OrderStatusOutbox + admin REST endpoint.
  - PENDING → INVENTORY_RESERVED → PAID → SHIPPED → CONFIRMED / FAILED / CANCELLED
- **inventory**: Event Sourcing + Spring Batch worker (Q7, ADR-0007).
  - `@Profile("worker")` Batch Job + chunk-oriented ItemReader/Processor/Writer.

### Istio Service Mesh

- **R-03 Istio platform 배포** (R-35 (A) / PR #81 / 1.23.3). istio-base + istiod + Gateway + VS + PeerAuthentication STRICT + DestinationRule.
- **R-62 istio-cp Application 분리** (PR #118). 12 cycle 동안 재발한 `ImagePullBackOff` 의 진짜 root cause = multi-source 안 chart 간 race. `platform/13-istio-ingressgateway/` 별도 Application (Wave -1) 으로 분리 후 영구 fix.
- **ADR-0009 책임 분담**: 외부 진입 (north-south) = Istio ingressgateway / 내부 통신 (east-west) = sidecar mTLS / 응용 라우팅 = api-gateway BFF + SCG / 인프라 retries = Istio / 응용 fallback = Resilience4j / JWT = api-gateway filter.

### Infra

- **AWS OIDC** — GitHub Actions → AWS, no long-lived access key.
  - Trust policy: `repo:KTCloud-CloudNative-Troica-Team/msa-*:ref:refs/heads/main`
- **VPC Endpoint × 3** — kubelet (private subnet) → ECR private repo 이미지 pull. ecr.api / ecr.dkr Interface (~$14/월) + S3 Gateway endpoint (무료).
- **Terraform backend** — S3 + native lockfile (Terraform 1.10+, `use_lockfile=true`). DynamoDB 불필요.
- **NLB** — k8s API (6443) + Istio Gateway (80/443 → NodePort 30080/30443, R-35 (d) / PR #22). source IP preservation 측 SG 의 `30080/30443 from 0.0.0.0/0` ingress 필수.
- **EC2 사양** — master 3 t3.medium + worker 3 t3.large (R-59 메모리 상향) + bastion 2 t3.nano.
- **self-managed kubeadm + Calico CNI IPIP mode** — IRSA 부재 → ESO / EBS CSI 등은 IMDS 인증 패턴.

### CI/CD

- **Trivy action v0.36.0** pinned (R-14: 공급망 사고 대응).
- **gradlew mode 100755** in git index (Windows 에서 만들면 100644, Linux CI 측 Permission denied).
- **AWS_DEPLOYMENTS_ENABLED Org Variable gate** (R-19) — 6 polyrepo CI 의 ECR push step 활성.
- **E2E_ENABLED Org Variable** (R-42) — msa-argocd-manifest 의 e2e-newman workflow 의 cluster 실 호출 활성.
- **manifest auto-bump** — CI 가 `values-dev.yaml` 의 `image.tag` 자동 commit (dev = direct, prod = PR 승인 게이트).

## 4. 활성 R 항목 (요약 — 자세한 건 BACKLOG)

### Phase 5 종료 시점 (2026-05-18) 측 미해결 — 모두 Phase 6 후속

| 항목 | 내용 | 상태 | 평가 영향 |
|---|---|---|---|
| R-50 (B) | RateLimit burst 시연 (anonymous fallback key 정정) | 📌 Phase 6 | (선택, 평가 영향 X) |
| R-51 | KEDA (Kafka lag → 자동 확장) | 📌 Phase 6 | 심화 (1)-4 (선택) |
| R-52 | ArgoCD Rollouts Canary | 📌 Phase 6 | 심화 (1)-5 (선택) |
| R-53 | OPA Gatekeeper | 📌 Phase 6 | 심화 (2)-5 (선택) |
| R-54 | OWASP ZAP DAST + SARIF | 📌 Phase 6 | 심화 (2)-4 (선택) |
| R-55 | Falco DaemonSet | 📌 Phase 6 | 심화 (3)-5 (선택) |
| R-56 | Incident Response 5 단계 자동화 | 📌 Phase 6 | 심화 (3)-4 (선택) |
| R-58 | Testcontainers 통합 테스트 | 📌 Phase 6 | 기본 (3)-4 (선택) |
| R-61 | OutOfSync 7 app `ignoreDifferences` 정리 | 📌 Phase 6 | 평가 영향 X (cosmetic) |
| R-63 | terraform `aws_instance.security_groups` → `vpc_security_group_ids` | 📌 Phase 6 | 평가 영향 X (in-place update 회피) |
| R-64 | platform/30-kube-prometheus-stack multi-source (troica dashboards 자동 sync) | 📌 Phase 6 | 평가 영향 X (현재 manual apply 운용) |
| R-40 | msa-frontend 정적 호스팅 (S3 + CloudFront + Route 53 + ACM) | ⏸ Phase 6 | 운영 전환 |
| R-08 | msa-spring-boot 모노레포 archive | ⏸ Phase 6 | 운영 전환 |
| R-15 | GitHub Actions SHA pin | ⏸ Phase 6 | 운영 전환 |
| R-18 | Spring Boot 3.5 OSS EOL 2026-06-30 모니터링 | ⏸ Phase 6 | 운영 전환 |

## 5. 핵심 디버깅 사례 (재발 방지용 + 발표 차별화)

### istio-cp multi-source race (R-62, 12 cycle 재발)

**증상**: `istio-ingressgateway` pod `ImagePullBackOff` 가 매 cluster destroy + apply 마다 재발.

**처음 가설 (틀린 fix)**:
- PR #111: istio-system namespace inject label 추가 → 같은 issue
- PR #117: istio-cp Sync Wave -4 → -2 → 같은 issue

**진짜 root cause**: multi-source helm 의 두 chart (istiod + gateway) 가 **같은 Sync Wave 안에서 동시 apply** → istiod pod Ready 전에 ingressgateway pod 첫 생성 시도 → webhook endpoint 비어 있음 → image `auto` stale pod → 영구 lock. 이전 PR 들 = Application 부모 ordering 만 해결. 진짜 race = 같은 Application 안 chart 간 ordering (Sync Wave 가 못 강제).

**영구 fix (PR #118)**: `platform/13-istio-ingressgateway/` 별도 Application (Wave -1) 분리.

### micrometer-registry-prometheus 의존성 부재 (R-65, 다중 layer fix)

**증상**: PR #120 (chart env `MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE`) 머지 → 새 pod 적용 → 단 `/actuator/prometheus` 여전히 404.

**root cause**: env 활성화해도 의존성 부재 → endpoint 자체 없음. **한 layer 만 만지면 효과 0**.

**영구 fix**: 6 polyrepo PR (api-gateway #14, auth #9, user #7, product #23, inventory #8, order #12) 의 build.gradle.kts 측 `runtimeOnly("io.micrometer:micrometer-registry-prometheus")` 추가.

### NLB Istio path 3 layer (R-35 (d) / PR #22)

**증상**: NLB DNS:80 외부 호출 → timeout.

**fact 추적 3 layer**:
1. NLB listener 측 6443 만 (Istio 측 누락) → terraform 측 80/443 listener + target group + attachment 추가
2. NLB source IP preservation = true → SG `30080/30443 from 0.0.0.0/0` ingress 필요 → `aws_security_group_rule` 추가
3. VirtualService 4 prefix 측 `/api/v1/, /admin/v1/orders, /healthz, /actuator/prometheus` 만 매칭

= 한 layer 만 봐서는 못 함. 평가 path 의 외부 진입은 layer 3 stack.

### terraform `aws_instance.security_groups` 함정 (R-63)

**증상**: SG rule 추가 (PR #22) 만 해도 `terraform plan` 이 **8 instance + 12 PVC 전부 force replacement** 표시.

**원인**: `aws_instance.security_groups` 는 EC2-Classic 시대 attribute (SG name 받음). 우리 코드가 SG ID 줌 → state 의 `security_groups` 는 empty + 실 attachment 는 `vpc_security_group_ids` 측. terraform plan 측 desired vs state diff = force replacement.

**임시 회복**: AWS CLI `authorize-security-group-ingress` 두 줄로 직접 추가.

**영구 fix (Phase 6 후속, R-63)**: 모든 `aws_instance.security_groups` → `vpc_security_group_ids` 교체 + SG inline ingress 모두 separate `aws_security_group_rule` 로 통일.

## 6. 다음 cluster 재배포 시 자동 작동 fact

모든 영구 fix 가 git main 측 머지 — 다음 cluster destroy + apply cycle 시 자동 적용:
- Istio path (R-62 분리, R-22 NLB Istio listener) → istio-ingressgateway 자동 Running
- chart env + 6 polyrepo micrometer 의존성 (R-65) → `/actuator/prometheus` 200
- Newman collection skip 로직 (PR #130) → 5/5 시나리오 PASS

ECR 이미지는 그대로 보존 → 서비스 재빌드 불필요. msa-argocd-manifest 의 매니페스트도 그대로 → ArgoCD 가 자동 sync.

---

*Last updated: 2026-05-18 — Phase 5 종료 (평가 18/18 완성). Phase 6 진입.*

---

## Historical Snapshots (Archive)

### 2026-05-12 — Phase 0 진행 중

PR 3 (S3 backend + prevent_destroy) 작업 중. 영구 17 자원 + 임시 ~45 자원 분류 + S3 backend bootstrap 측 README 작성. 비용 사이클 측 destroy-temp.sh 작성. Spring Boot 3.5.13 + Kotlin 2.1.0 + Gradle 8.10.2 + Kafka wire JSON 결정. common-libs v0.3.0 publish (2 모듈). order state machine 7-state + inventory Event Sourcing 결정.

### 2026-05-14 — Phase 5 매니페스트 작업

R-35 매니페스트 22 파일 일괄 작성 (cert-manager → external-secrets-config). cluster up 전 사전 작성. AWS Secrets Manager 9 secrets 등록 + Slack `#security-report` 채널 생성 + EC2 node IAM role 의 `SecretsManager:GetSecretValue` 권한 추가 측 사용자 사전 작업. 평가 cover = ✅ 13 / 🔄 3 / ⏸ 2.

### 2026-05-17 — Cluster up + 평가 검증 (12/14 86%)

Cluster 안정화 누적 fix 15 PR 머지 (worker root volume / AWS LB Controller 제거 / ArgoCD nodePort 30090/30493 / manifest diet / Spotahome redis-operator 제거 / Istio namespace inject label / AlertManager secrets 들여쓰기 fix). 평가 검증 12/14 = R-42 외 모두 통과. cluster destroy + 재배포 후 R-42 진행 예정.
