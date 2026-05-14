# Troica MSA 작업 백로그 (Phase별)

> **참조**: 모든 작업은 [TROICA_SPEC.md](./TROICA_SPEC.md)의 Phase 구조를 따른다.
> 트러블슈팅 세부 내용은 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) 참조 (본 문서는 task 진행 상태만).
>
> **작업 원칙**
> - `main` 직접 커밋 금지. 모든 변경은 feature branch + PR.
> - **`msa-spring-boot` 레포는 read-only**. 다른 작업자가 main에서 작업 중이므로 push 금지.
> - Phase 단위로 진행하며 §14 검증 체크리스트를 통과해야 다음 Phase 진입.

## 상태 범례

| 기호 | 상태 |
|---|---|
| ✅ | 완료 (PR 머지) |
| 🔄 | 진행 중 (PR 작업 중 또는 머지 대기) |
| ⏸ | 대기 (선결 조건 필요) |
| 🔍 | 검토 필요 |
| 📌 | 보류 / 미래 작업 |
| ❌ | 폐기 (작업하지 않기로 결정) |

## ID convention

본 문서 + commit message + PR 본문에 사용된 ID prefix는 역사적으로 다음 의미로 사용됨:

| prefix | 의미 | 예시 | 현재 정책 |
|---|---|---|---|
| **R-NN** | Risk / follow-up / task | R-27 (CI 최적화), R-38 (kotlin-spring) | ✅ **앞으로 신규 task는 모두 R-NN으로 통일** |
| **P*<phase>.<seq>* | Phase 작업 단위 | P0.1, P3.2 | 📌 historical — 현재는 표의 Phase 섹션으로 분류 충분 |
| **D*N*** | 누적 결정 (Decision) | D1 ~ D7 | 📌 historical — 모두 [ADR](./adr/)로 흡수됨, 본 문서는 링크만 유지 |
| **Q*N*** | 회의 검토 질문 (Question) | Q1 ~ Q7 | 📌 historical — D와 동일하게 ADR로 흡수 |
| **P1-V** | 특수 (Phase 1 검증) | (1회만 사용) | 📌 historical |

**규칙**:
- ✏️ **신규 task는 R-NN 사용**. 다음 번호는 본 문서의 마지막 R 번호 + 1.
- 📜 **과거 P/D/Q는 변경 안 함** — commit history + PR 본문에 이미 흩뿌려져 있어 immutable.
- 🔗 **D/Q 의사결정 자체는 [docs/adr/](./adr/)이 단일 진실**. 본 문서는 해당 ADR에 링크만.
- 🔁 향후 Phase 6 (정리/발표) 시점에 전면 재구성 검토 가능.

---

## Phase 0 — AWS 인프라 + ECR + 부트스트랩 (완료)

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| P0.1 | Terraform OIDC + ECR repo × 6 + VPC Endpoint + KMS | ✅ | `msa-provisioning phase-0/aws-ecr-oidc` 머지 | 영구 자원 |
| P0.2 | kubelet ECR credential provider — Ansible playbook 골격 | ✅ | `msa-provisioning phase-0/kubelet-ecr-credential-provider` 머지 | |
| P0.3 | S3 backend + 영구/임시 자원 분리 (`prevent_destroy`) | ✅ | `msa-provisioning phase-0/permanent-resources-and-backend` 머지 | 비용 사이클 가능 |
| P0.4 | README S3 부트스트랩 절차 fix | ✅ | `msa-provisioning chore/readme-s3-bootstrap-fix` 머지 | `RestrictPublicBuckets` + PowerShell 예제 |
| R-19 | CI push-gated step `AWS_DEPLOYMENTS_ENABLED` gate | ✅ | SPEC §7 + 6개 서비스 ci.yml에 가드 추가 | Org Variable 등록 시 자동 활성 |
| R-26 | 매니페스트 `image.repository` placeholder → 실제 account_id | ✅ | `msa-argocd-manifest phase-0/manifest-account-id` 머지 | 6개 서비스 values.yaml 일괄 sed |
| R-29 | cloud-provider-aws ecr-credential-provider 바이너리 빌드 패턴 | ✅ | `msa-provisioning fix/ansible-ecr-credential-provider-integrated` 머지 | Go 빌드 + ansible local→remote copy. 자세한 절차는 playbook vars 주석 |
| R-30 | kubelet `KUBELET_EXTRA_ARGS` drop-in 미평가 → `kubeadm-flags.env` 직접 통합 | ✅ | 같은 PR | 근본 원인 미규명 (Phase 6 발표 후 추적) |
| R-31 | bootstrap/root-app.yaml app-of-apps 정식 패턴 | ✅ | `msa-argocd-manifest fix/r-31-root-app-app-of-apps` 머지 | brace expansion `{a,b,c}` ArgoCD 미지원 |
| R-32 | ansible argocd-setup.yaml의 root-app repo URL 우리 팀 레포로 | ✅ | `msa-provisioning fix/r-32-argocd-setup-repo-url` 머지 | inline yaml → fetch+apply 패턴 |
| R-34 | `destroy-temp.sh` Windows 호환 | ✅ | `chore/destroy-temp-powershell` + `chore/gitattributes-lf-enforcement` 머지 | PowerShell native 추가 + `.sh` LF 강제 |
| R-37 | platform/root.yaml brace expansion (R-31 동일 패턴) | ✅ | `msa-argocd-manifest fix/r-37-platform-root-glob` 머지 | `**/application.yaml` doublestar |

**Phase 0 검증 완료**:
- terraform apply/destroy 사이클 작동
- 6개 서비스 ECR push 성공 (Trivy 12개 HIGH CVE 차단 + 해결 후)
- ArgoCD가 ApplicationSet으로 12개 자식 app 자동 생성
- kubelet credential provider 정상 (`Successfully pulled image` 검증)

---

## Phase 1 — ArgoCD 매니페스트 골격 (완료)

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| P1.1 | `TROICA_SPEC.md` 청사진 + `BACKLOG.md` 신설 | ✅ | `bootstrap/spec-and-backlog` 머지 | |
| P1.2 | 매니페스트 디렉토리 구조 (bootstrap, projects, applications, platform) | ✅ | `phase-1/manifest-skeleton` 머지 | |
| P1.3 | Helm chart `microservice` (deployment, service, hpa, sa, configmap, virtualservice) | ✅ | 동일 PR | `helm lint` + `helm template` 통과 |
| R-01 | `values-worker.yaml` 별도 파일 미사용 → SPEC 정정 | ✅ | SPEC §2 dead file 제거 | |
| R-02 | App-of-Apps multi-source 스니펫 오류 | ✅ | SPEC §6 `source:+sources:` 중복 정리 | |
| P1-V | 클러스터-사이드 검증 (`kubectl apply --dry-run=server`) | ✅ | Phase 0 클러스터에서 검증 완료 | |

---

## Phase 2 — common-libs 멀티모듈 + Protobuf (완료)

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| P2.1 | 4-module Gradle 프로젝트 (common, client-redis, client-ses, events) | ✅ | `msa-common-libs phase-2/initial-setup` 머지 | |
| P2.2 | Protobuf 코드젠 + Kotlin codegen | ✅ | 동일 PR | order/notification 도메인 proto |
| P2.3 | GitHub Packages publish workflow | ✅ | 동일 PR | tag push 시 `publish.yml` |
| P2.4 | gradlew 100755 fix | ✅ | `phase-2/initial-setup` 추가 commit | Windows에서 만든 gradlew 권한 fix |
| R-09 | Kotlin/Gradle 버전 충돌 (Kotlin 2.3.x → 2.1.0) | ✅ | toolchain 통일 (2.1.0 + 8.10.2) | JetBrains 버그 (R-16 동일 사안) |
| R-10 | 이벤트 스키마 Protobuf vs Avro → Protobuf 단독 | ✅ | SPEC §9.3 재작성 | |
| R-11 | common-libs GH Packages publish 실 검증 | ✅ | v0.1.0/v0.2.0/v0.3.0 tag push 검증 | |
| D2 | client-redis JitPack (`com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`) | ✅ | common-libs v0.3.0에서 client-redis 모듈 제거 | 팀장님 외부 publish 사용 |
| D3 | client-ses 제거 (notification 폐기 일관) | ✅ | common-libs v0.3.0 | |

---

## Phase 3 — order-service polyrepo + worker (완료)

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| P3.1 | 2-module Gradle (order, order-service) | ✅ | `msa-order-service phase-3/initial-setup` 머지 | 모노레포에서 추출 |
| P3.2 | order-worker `@Profile("worker")` + `@Scheduled` Outbox poller | ✅ | 동일 PR | R-07 해소 (worker 분기 신규 1 파일) |
| P3.3 | order-service values 매니페스트 추가 (dev/prod) | ✅ | `phase-3/order-service-values` 머지 | api + worker 2 Deployment |
| P3.4 | Tech stack audit + Spring Boot 3.3.0 → 3.5.13 upgrade | ✅ | `upgrade/spring-boot-3.5` (common-libs + order-service) 머지 | CVE-2025-22235 fix + EOL 회피 |
| R-06 | Spring Boot 버전 + JarLauncher 경로 검증 | ✅ | bootJar MANIFEST 확인 | |
| R-07 | order-worker profile 분기 코드 | ✅ | 신규 1 파일로 해소 | |
| R-12 | 로컬 Docker build 미검증 | ✅ | Phase 0 CI에서 6개 서비스 모두 ECR push 성공으로 검증됨 | |
| R-14 | Trivy action 0.24.0 미존재 + 공급망 사고 | ✅ | `@v0.36.0` (post-incident) pin | |
| R-17 | common-libs v0.2.0 publish 후 deps 재bump | ✅ | 자연스럽게 0.3.0까지 진행됨 | |

---

## Phase 4 — 6개 서비스 polyrepo + 프론트엔드 (완료)

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| P4.1 | msa-product-service | ✅ | `phase-4/initial-setup` 머지 | 단순 추출 |
| P4.2 | msa-order-service 토픽명 + state machine 확장 (Q5/Q6) | ✅ | `phase-4/topics-and-state-machine` 머지 | `OrderLineItemStatus` 확장 + OrderStatusOutbox + admin endpoint |
| P4.3 | msa-inventory-service Event Sourcing + Spring Batch worker (Q7) | ✅ | `phase-4/initial-setup` 머지 | `@Profile("worker")` Batch Job/Step |
| P4.4 | msa-user-service (user lib publish + user-service app) | ✅ | `phase-4/initial-setup` 머지 | dual delivery 패턴 |
| P4.5 | msa-auth-service 신규 레포 (D1) | ✅ | `phase-4/initial-setup` 머지 | gRPC `AuthService.{SignUp, SignIn, CheckValidity}` |
| P4.6 | msa-api-gateway 단일 모듈 BFF + Gateway 혼합 (Q1 c) | ✅ | `phase-4/initial-setup` 머지 | Spring Cloud 2025.0.2 Northfields |
| P4.7 | 6개 매니페스트 values (dev/prod) | ✅ | argocd-manifest 6개 PR 머지 | `applications/values/<service>/` |
| R-39 | **msa-frontend 신규 레포** (팀장님 작성) | ✅ | `msa-frontend/market-msa-app/` | **React 19.2 + Vite 8 + TypeScript 6 + TanStack Router/Query + MUI v9 + Tailwind v4 + Zustand**. 백엔드와 별도 polyrepo (10번째). S3+CloudFront 정적 호스팅은 Phase 6 |
| R-04 | api-gateway 명칭 결정 | ✅ | `msa-api-gateway` 신규 레포 | Q1 (c) 혼합 |
| R-13 | Kafka JSON 유지 (Protobuf 마이그레이션 보류) | ✅ | Q4 (a) JSON wire 유지 | 추후 Protobuf 전환은 R-13 잔존 |
| R-21 | Q6 order state machine 확장 | ✅ | P4.2와 동일 | |
| R-22 | Q7 inventory Event Sourcing + Spring Batch | ✅ | P4.3과 동일 | |
| R-23 | api-gateway BFF + SC Gateway 라우팅 매트릭스 | ✅ | application.yaml에 명시 | BFF gRPC clients + SC Gateway routes |
| R-24 | identification 모듈 폐기에 따른 user/auth 영향 | ✅ | identification 의존 흔적 제거 확인 | |
| D1 | auth-service 신규 레포 + notification 폐기 | ✅ | msa-auth-service 신규 + msa-notification-service archive | |
| D6 | notification-service archive | ✅ | GitHub UI archive | |
| R-25 | auth-service 평문 JWT secret | 🔄 | `fix/r-25-jwt-secret-placeholder` (1차 fix). Phase 5에서 ExternalSecrets로 정식 해결 | placeholder fallback으로 hygiene |

---

## Phase 5 — Platform 배포 + 운영 강화 + 평가요소 100% (대기)

> **평가요소 cover**: 본 Phase는 KT Cloud Tech UP 2기 발표 평가요소 (기본·심화 필수 항목)를 100% 충족시키기 위한 작업을 포함한다. 각 task의 비고에 `평가 기본/심화 (N)-M` 형식으로 매핑.

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| R-03 | Traefik → Istio 교체 | ⏸ | Phase 5 본 작업 | Istio Gateway + VirtualService. **평가 심화 (1)-2 일부** |
| R-33 | 6×2 Secret + ConfigMap 실값 | ⏸ | ExternalSecrets Operator + AWS Secrets Manager (R-25 정식 해결과 묶음) | |
| R-35 | Platform 배포 (a~g) | ⏸ | (a) PostgreSQL × 6 (b) Redis × 2 (c) Strimzi Kafka + KafkaTopic × 4 (d) Istio (e) AlertManager + Slack (f) Prometheus/Grafana/Loki (g) ExternalSecrets | Phase 5 본 작업, 비용 영향 大. **평가 기본 (1)-4 (Kafka StatefulSet)** |
| **R-41** | **K8s 디스커버리 검증 + 장애 격리 시나리오 코드** | ⏸ | (1) ClusterIP+DNS 서비스 간 통신 검증 task (`kubectl exec` + nslookup + curl) (2) Response Aggregate Fallback 코드 — order-service가 inventory 다운 시 부분 응답 반환 + 단위 테스트 | **평가 기본 (2)-1, (2)-3**. PROJECT_PLAN §6.8 문서를 코드로 구현 |
| **R-42** | **테스트 자동화 + Observability** | ⏸ | (1) Postman Collection (회원가입→주문생성→재고예약→확정 E2E) (2) Newman CI 단계 추가 (각 polyrepo `.github/workflows/ci.yml` 또는 별도 e2e workflow) (3) Prometheus + Grafana 대시보드 (서비스별 에러율 + P99 latency) | **평가 기본 (3)-2, (3)-3** + 선택 (3)-5. R-35 (f) 와 연계 |
| **R-43** | **PodDisruptionBudget templates** | ✅ | `applications/charts/microservice/templates/pdb.yaml` 신설 + `values.yaml`에 `pdb.{enabled,minAvailable}` 기본값. api: minAvailable=1 (replicaCount=2 기준). worker: maxUnavailable=1 (replicaCount=1 drain 가능). prod values에서 minAvailable=2 override 권장 | **평가 심화 (1)-3**. drain 시뮬레이션 검증은 cluster up 후 R-42와 함께 |
| **R-44** | **독립 배포 E2E + API GW ↔ Service Mesh 협업 검토** | 🔄 | (1) 한 서비스만 image bump → 다른 서비스 트래픽 무영향 E2E 검증 (Newman) — Phase 5 R-42와 함께 ⏸ (2) api-gateway (BFF + SC Gateway) + Istio (mTLS + VirtualService) 책임 분담 ADR ✅ [ADR-0009](./adr/0009-api-gateway-istio-mesh-collaboration.md) | **평가 심화 (1)-1 ⏸ / (1)-2 ✅**. ADR-0009 작성 완료 |
| **R-45** | **SonarCloud (public 무료) CI 통합 + 커버리지 게이트** | ⏸ | (1) 6 polyrepo CI에 SonarCloud scan step 추가 — **public repo는 무료 plan 사용 (비용 $0)** (2) 커버리지 80% 미만 또는 Critical 이슈 발견 시 fail (3) `sonarqube-action` PR 코멘트 자동 게시 | **평가 심화 (2)-1**. Gradle JaCoCo 플러그인 연계. 모든 polyrepo가 org public 레포 → SonarCloud public plan 적합 |
| **R-46** | **Trivy config (Kubernetes 매니페스트 스캔)** | ✅ | 본 레포 `.github/workflows/trivy-manifest-scan.yml` 신설 — `applications/`, `platform/`, `bootstrap/`, `projects/` 스캔, SARIF GitHub Security 탭 업로드, 실패 시 Slack `#security-report` webhook 발송 | **평가 심화 (2)-2**. PR 트리거 + main push 트리거 |
| **R-47** | **Slack `#security-report` 보안 알림 채널** | 🔄 | (1) ADR-0010 작성 ✅ — 채널 분리 결정 + AlertManager 라우팅 예시 (2) Trivy manifest scan CI에 Slack webhook step ✅ (R-46 워크플로 내) (3) AlertManager 실 매니페스트는 R-35 (e)와 함께 ⏸ (4) Slack 채널 생성 + webhook 발급은 Phase 5 진입 시 수동 | **평가 심화 (2)-3**. CI 절반 + ADR ✅, AlertManager 절반 ⏸ |
| **R-48** | **K8s RBAC (사람 + 서비스 양쪽) 역할 분리** | ✅ | (1) **사람 RBAC**: `platform/05-rbac/` 신설 — developer(조회 ClusterRole) / operator(market-{dev,prod} Role) / sre(cluster-admin 바인딩) Group binding. Sync Wave -8 (2) **서비스 RBAC**: Helm 차트 `templates/role.yaml` 신설 — 서비스별 Role + RoleBinding, 기본 `rules: []` (최소 권한). OIDC provider 통합은 Phase 6 | **평가 심화 (3)-1, (3)-2**. Group binding subjects는 OIDC 통합 후 active |
| **R-49** | **NetworkPolicy + 통신 차단 검증** | ✅ | (1) `platform/60-network-policies/` 신설, Sync Wave 5 (2) market-{dev,prod} 각각: default-deny-ingress + api-gateway←Istio + backends←api-gateway + user/order-admin←api-gateway + prometheus scrape 허용 (3) 검증 절차: `kubectl exec` 다른 서비스에서 curl 거부 확인 (cluster up 후 R-42와 함께) | **평가 심화 (3)-3**. Calico CNI는 NetworkPolicy enforce ✅ |
| R-27 (a) | Gradle 중복 빌드 제거 (호스트 + Docker → Docker 단순화) | ✅ | 6 PR 머지 (product/auth/user/order/inventory/api-gateway의 `chore/r-27-*`) | 빌드 시간 ~5분 → **2분 25초** 검증 (product-service). Docker 3-stage → 2-stage, 호스트 bootJar 결과를 layered extract만. 부가: glibc base 불필요 (alpine만) → TROUBLESHOOTING §3.1 (Alpine musl) 자동 회피 |
| R-27 (e) | `paths-ignore` (docs PR build skip) | ✅ | `msa-product-service chore/r-27-paths-ignore-and-dependabot` 머지 | `**.md`, `docs/**`, `LICENSE`, `.gitignore`, `.gitattributes` skip. 5개 서비스 + common-libs propagate는 Phase 5 후 (영향 작음) |
| R-27 (g) | Dependabot 활성화 | ❌ 폐기 | `msa-product-service chore/remove-dependabot` 머지 (역삭제) | 첫 주에 7+개 자동 PR 폭주 → review 부담. Spring Boot BOM이 대부분 dep 관리 + Gradle bump가 Kotlin 2.1 toolchain 결정과 충돌. 추후 Renovate 또는 monthly check 같은 가벼운 대안 검토 |
| R-27 (b/c/d/f) | BuildKit cache / Trivy DB cache / Reusable workflow / ECR retry | 📌 | (b) Docker BuildKit cache mount → -30~60s/2nd+ 빌드 (c) Trivy cache key 주 단위 → -1분/매일 첫 (d) 6 ci.yml 통합 (e) ECR login retry | Phase 5 검증 끝나면 일괄. (a) 검증 후 우선순위 낮음 |
| R-28 | netty 4.2.x BOM 전환 → CVE-2026-42577 해제 | 📌 | Spring Boot 3.5.15+ 또는 reactor-netty 1.3.x 출시 모니터링 | 외부 의존성 |
| R-20 | JitPack client-redis 모니터링 | 📌 | 버전 변경 알림 (monthly check) | 팀장님 통제 외, Dependabot 폐기됨 |
| R-38 | common-libs `QuerydslConfig` final → Spring CGLIB proxy 실패 | ✅ | `msa-common-libs fix/r-38-kotlin-spring-plugin` 머지 + v0.3.1 publish + 6 서비스 의존성 bump | Kotlin은 모든 class 기본 final → @Configuration proxy 못 만듦. common/build.gradle.kts에 `kotlin("plugin.spring")` 추가로 자동 open. Phase 0 demo 시 CrashLoop의 진짜 원인이었음 |

---

## Phase 6 — 정리 + 발표 (대기)

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| R-08 | `msa-spring-boot` 모노레포 archive | ⏸ | GitHub UI archive | 모든 서비스 이전 완료 후 |
| R-15 | GitHub Actions SHA pin + Dependabot | ⏸ | 보안 강화 트랙 | Phase 6 또는 별도 |
| R-18 | Spring Boot 3.5 OSS EOL 2026-06-30 모니터링 | ⏸ | Phase 6 종료 시점 또는 EOL 2주 전 검토 | 3.6.x 또는 4.0 마이그레이션 |
| R-40 | **msa-frontend 정적 호스팅 배포** (S3 + CloudFront + Route 53 + ACM) | ⏸ | terraform 모듈 + CI 빌드 산출물 S3 sync | E8 epic. msa-provisioning에 frontend 호스팅 자원 추가 |
| R-50 | **(선택 평가요소) Rate Limit 실 구현** | ⏸ | api-gateway `application.yaml`에 SC Gateway `RequestRateLimiter` filter + Redis (분당 token bucket) | **평가 기본 (2)-5 선택**. 백로그/PLAN에는 명시되어 있으나 코드 미반영 |
| R-51 | **(선택 평가요소) KEDA (Kafka lag 기반 ScaledObject)** | 📌 | inventory consumer가 `order.pending` lag 임계값 도달 시 Pod 자동 확장 | **평가 심화 (1)-4 선택**. 발표 데모 임팩트 큼 |
| R-52 | **(선택 평가요소) ArgoCD Rollouts Canary** | 📌 | order-service 등 1개 서비스에 Canary 단계별 분기 + 자동 롤백 | **평가 심화 (1)-5 선택**. ArgoCD 이미 설치되어 비용 작음 |
| R-53 | **(선택 평가요소) OPA Gatekeeper** | 📌 | ConstraintTemplate (`runAsNonRoot: true`, `readOnlyRootFilesystem: true`) → Namespace 전체 강제 + 정책 위반 거부 테스트 | **평가 심화 (2)-5 선택** |
| R-54 | **(선택 평가요소) OWASP ZAP DAST + SARIF** | 📌 | staging에서 ZAP scan → SARIF GitHub Security 탭 업로드 | **평가 심화 (2)-4 선택** |
| R-55 | **(선택 평가요소) Falco DaemonSet 런타임 탐지** | 📌 | 비정상 shell 실행 등 이벤트 Slack `#security-report` 연동 | **평가 심화 (3)-5 선택** |
| R-56 | **(선택 평가요소) Incident Response 5단계 자동화** | 📌 | 탐지 → 격리 → 분석 → 복구 → 회고 runbook + 각 단계 스크립트 | **평가 심화 (3)-4 선택** |
| P6.1 | 발표 자료 (architecture diagram + decision log + 비용 분석) | ⏸ | KT Cloud Tech UP 2기 발표 | |

---

## 비-Phase / 코드 품질

| ID | Task | 상태 | 산출물 | 비고 |
|---|---|---|---|---|
| R-05 | notification-service 그린필드 | ✅ | D6에 의해 폐기 (archive) | |
| R-16 | Kotlin 2.3.x + Gradle 9.x JetBrains 버그 모니터링 | 📌 | R-09와 동일. 2.4.x 또는 2.3.22+ release 후 재시도 | |
| R-36 | ansible playbook + README 일본어 → 한국어 | ✅ | `i18n/ansible-japanese-to-korean` 머지 | 14개 파일 일괄 |

---

## SPEC 변경 이력

본 섹션은 SPEC.md의 누적 변경 기록. (자세한 변경 사유는 git log + 본 BACKLOG 또는 TROUBLESHOOTING 참조)

| 일자 | SPEC 위치 | 변경 | 트리거 R |
|------|----------|------|----------|
| 2026-05-12 | §2 디렉토리 트리 | `values-worker.yaml` 제거, worker 분기 = flag | R-01 |
| 2026-05-12 | §6 multi-source 예시 | `source:+sources:` 중복 정리 | R-02 |
| 2026-05-12 | §0 결정 사항 표 | Protobuf, Istio 행 추가 | R-10, R-03 |
| 2026-05-12 | §1.2 공용 라이브러리 | 멀티모듈 4 서브모듈 명시 | Phase 2 |
| 2026-05-12 | §9.3 페이로드 스키마 | "Protobuf 또는 Avro" → "Protobuf only + no registry for PoC" | R-10 |
| 2026-05-12 | §12 msa-common-libs | 단일 → 멀티모듈 publish 패턴 + 빌드환경 § | Phase 2 |
| 2026-05-12 | §13.2/§13.3 Phase 작업 순서 | 단계 확장 (멀티모듈, Protobuf codegen, worker 신규, gradlew 100755) | Phase 2/3 |
| 2026-05-12 | §7 CI Trivy 버전 | `0.24.0` → `v0.36.0` + 공급망 사고 주석 | R-14 |
| 2026-05-12 | §0/§12.0.1 | SB `3.3.0` → `3.5.13`, Spring Cloud 2025.0.2 명시 | CVE-2025-22235 + Q1 |
| 2026-05-12 | §13.3 step 7 | "SB 3.3.0 검증" → "SB 3.5.13 검증" | upgrade 결과 |
| 2026-05-12 | §7 CI + §7.1 Secrets 표 | `AWS_DEPLOYMENTS_ENABLED` 게이트 + Variable 행 신설 | R-19 |

---

## 평가요소 매핑 (KT Cloud Tech UP 2기 발표)

본 섹션은 발표 평가요소 → BACKLOG task 역인덱스. **필수 항목은 모두 R-NN 매핑되어야 함** (불일치 시 신규 R 추가).

### 기본 프로젝트

| 평가요소 | 필수/선택 | 대응 R | 상태 |
|---|---|---|---|
| (1)-1 이벤트 스토밍 + 독립 배포/확장/격리 | 필수 | P1.x / Phase 4 polyrepo 전체 | ✅ |
| (1)-2 동기 REST vs 비동기 이벤트 적용 기준 | 필수 | ADR-0003 / ADR-0005 / PROJECT_PLAN §5.2 | ✅ |
| (1)-3 Database per Service | 필수 | Phase 4 전체 + PROJECT_PLAN §5.3 | ✅ |
| (1)-4 Kafka StatefulSet + 토픽 + 비동기 | 선택 | R-35 (c) Strimzi + ADR-0004 | ⏸ |
| (1)-5 Event Sourcing 1개 서비스 | 선택 | R-22 / ADR-0007 (inventory) | ✅ |
| (2)-1 K8s ClusterIP + DNS 디스커버리 | 필수 | **R-41** | ⏸ |
| (2)-2 API GW + 경로 라우팅 + JWT 필터 | 필수 | ADR-0005 / P4.6 | ✅ |
| (2)-3 장애 격리 시나리오 설계 + 코드 | 필수 | **R-41** | ⏸ |
| (2)-4 Resilience4j + Fault Injection | 선택 | Epic E7 (PROJECT_PLAN) | ⏸ Phase 5 |
| (2)-5 Rate Limit | 선택 | **R-50** | 📌 |
| (3)-1 JUnit 단위 + CI | 필수 | Phase 0-4 CI 전체 | ✅ |
| (3)-2 Postman E2E (서비스 연계) | 필수 | **R-42** | ⏸ |
| (3)-3 Prometheus + Grafana | 필수 | **R-42** + R-35 (f) | ⏸ |
| (3)-4 Testcontainers | 선택 | PROJECT_PLAN §4.1 명시 (실 사용은 코드 점검) | 🔍 |
| (3)-5 Newman CLI in CI | 선택 | **R-42** | ⏸ |
| (3)-6 Kafka consumer lag Exporter | 선택 | Epic E12 + R-35 (f) | ⏸ Phase 5 |

### 심화 프로젝트

| 평가요소 | 필수/선택 | 대응 R | 상태 |
|---|---|---|---|
| (1)-1 독립 배포 + E2E 무영향 검증 | 필수 | **R-44** (검증은 Phase 5) | 🔄 |
| (1)-2 API GW ↔ Service Mesh 협업 | 필수 | **R-44** ADR-0009 + R-03 | ✅ |
| (1)-3 PodDisruptionBudget | 필수 | **R-43** | ✅ |
| (1)-4 KEDA | 선택 | **R-51** | 📌 |
| (1)-5 ArgoCD Rollouts Canary | 선택 | **R-52** | 📌 |
| (2)-1 SonarQube 80% 게이트 | 필수 | **R-45** | ⏸ |
| (2)-2 Trivy config (매니페스트 스캔) | 필수 | **R-46** | ✅ |
| (2)-3 Slack #security-report | 필수 | **R-47** ADR-0010 + R-46 CI webhook (AlertManager는 R-35 (e)와 함께) | 🔄 |
| (2)-4 OWASP ZAP DAST | 선택 | **R-54** | 📌 |
| (2)-5 OPA Gatekeeper | 선택 | **R-53** | 📌 |
| (3)-1 RBAC 역할 분리 | 필수 | **R-48** | ✅ |
| (3)-2 ServiceAccount + Role/RoleBinding | 필수 | **R-48** | ✅ |
| (3)-3 NetworkPolicy + 차단 테스트 | 필수 | **R-49** (검증은 Phase 5) | 🔄 |
| (3)-4 Incident Response 5단계 | 선택 | **R-56** | 📌 |
| (3)-5 Falco DaemonSet | 선택 | **R-55** | 📌 |

**필수 항목 (총 18개) 모두 R-NN 매핑 완료 ✅**
- 기본 필수 9 중 ✅ 6 완료 / ⏸ 3 (R-41/R-42)
- 심화 필수 9 중 ✅ 5 (R-43/R-46 + R-48 사람·서비스 + R-44 ADR) / 🔄 3 (R-44 검증, R-47 AlertManager, R-49 검증) / ⏸ 1 (R-45)

---

## 주요 의사결정 (ADR)

자세한 컨텍스트 / 대안 / 결과는 [docs/adr/](./adr/) 디렉토리의 각 ADR 참조.

| ADR | 결정 | 관련 R / Q / D |
|---|---|---|
| [ADR-0001](./adr/0001-polyrepo-with-auth-service.md) | Polyrepo 구조 + auth-service 분리 + notification 폐기 | D1, D6, Q2, Q3, R-05 |
| [ADR-0002](./adr/0002-client-libraries-distribution.md) | client-redis는 JitPack, client-ses 제거 | D2, D3, R-20 |
| [ADR-0003](./adr/0003-kafka-wire-format-json.md) | Kafka 직렬화 = JSON 유지 (Protobuf 마이그레이션 보류) | Q4, R-13 |
| [ADR-0004](./adr/0004-kafka-topic-naming.md) | Kafka 토픽명 = SPEC 표준 (`order.pending` 등) | Q5 |
| [ADR-0005](./adr/0005-api-gateway-bff-with-cloud-gateway.md) | api-gateway = BFF + Spring Cloud Gateway 혼합 | Q1, R-04, R-23 |
| [ADR-0006](./adr/0006-order-state-machine-extension.md) | order state machine 7-state 확장 | Q6, R-21 |
| [ADR-0007](./adr/0007-inventory-event-sourcing-batch-worker.md) | inventory = Event Sourcing + Spring Batch worker | Q7, R-22 |
| [ADR-0008](./adr/0008-order-terminal-events.md) | order.confirmed/cancelled 토픽 발행 유지 | D7 |
