# Troica MSA 작업 백로그

> **참조**: 모든 작업은 [TROICA_SPEC.md](./TROICA_SPEC.md)의 Phase 구조를 따른다.
> 결정 변경 시 SPEC도 함께 갱신하고 본 문서의 **SPEC 변경 이력** 섹션에 한 줄 기록한다.
>
> **작업 원칙**
> - `main` 직접 커밋 금지. 모든 변경은 feature branch + PR.
> - **`msa-spring-boot` 레포는 read-only**. 다른 작업자가 main에서 작업 중이므로 push 금지.
> - Phase 단위로 진행하며 §14 검증 체크리스트를 통과해야 다음 Phase 진입.
> - 결정 변경이 발생하면 SPEC.md 본문을 갱신하고 본 백로그에 변경 이력을 남긴다.

---

## 진행 중

- **Phase 0 — AWS 인프라** (2026-05-12 진행):
  - PR 1: `msa-provisioning phase-0/aws-ecr-oidc` — Terraform OIDC + 6 ECR repo + VPC Endpoint + KMS. terraform validate 통과
  - PR 2 (stacked): `msa-provisioning phase-0/kubelet-ecr-credential-provider` — node role ECR readonly attach + ansible playbook (credential-provider-config + kubelet drop-in)
  - 사용자 액션: PR 1 머지 → terraform apply → Org Secrets(`AWS_ACCOUNT_ID`, `MANIFEST_PAT`) + Variable(`AWS_DEPLOYMENTS_ENABLED=true`) 등록 → PR 2 머지 → ansible-playbook 실행
- **Phase 4 머지 진행 중** — auth-service, api-gateway 미머지. 다른 4개(product/order/inventory/user) 머지 완료.

---

## 팀장님 답변 (2026-05-12 수령) 및 후속 결정

팀장님 답변 후 모노레포 main 변경사항 점검 + 신규 결정 (D1~D6) 정리. SPEC v2.0 수준의 변경.

### 팀장님 P0~P2 답변

| # | 질문 | 답변 |
|---|------|------|
| Q1 | api-gateway 정체 | **(c) BFF + Spring Cloud Gateway 혼합 (한 서비스에 통합)** |
| Q2 | user/auth/identification 묶음 | **identification 폐기**, auth는 별도 서비스. 모노레포 main에 `auth-service` 신규 추가 |
| Q3 | notification scope | **notification-service 폐기**. Grafana → Slack 알림만 |
| Q4 | Kafka 직렬화 | **(a) JSON 유지** |
| Q5 | Kafka 토픽명 | **(a) SPEC 표준으로 정렬** |
| Q6 | order state machine | **(a) 확장 — 우리가 구현** |
| Q7 | inventory event sourcing + Spring Batch worker | **(b) — 우리가 구현** |

### 후속 결정 (D1~D6 — 2026-05-12 확정)

| # | 결정 | 의도 |
|---|------|------|
| D1 | `auth-service` → 신규 레포 `msa-auth-service` 생성. notification은 archive (D6) | 레포 1개당 1 deployable 원칙 일관성 |
| D2 | `client-redis`는 팀장님의 **JitPack** 패키지(`com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`)로 일원화. common-libs:client-redis는 v0.3.0에서 제거 | 팀장님이 이미 외부 publish 운영 중 — 중복 제거 |
| D3 | `client-ses` common-libs에서 제거 (notification 폐기와 일관) | dead code 정리 |
| D5 | Q6/Q7 모두 우리가 구현. Phase 4 진행하면서 함께 | 별도 Phase로 분리 안 함 |
| D6 | `msa-notification-service` → GitHub UI에서 **Archive** | delete 대신 read-only 보존 |
| **D7** (암묵) | `order.confirmed`/`order.cancelled` 토픽 발행은 유지하되 현재 consumer 없음 | Outbox 일관성 + 추후 subscribe 가능성 유지 |

### 사용자 대기 액션

- [ ] **GitHub UI**: `msa-notification-service` Archive (Settings → General → Danger Zone)
- [ ] **GitHub UI**: 신규 레포 `msa-auth-service` 생성 (LICENSE + README만)
- [ ] common-libs v0.3.0 PR 머지 후 `git tag v0.3.0 && git push origin v0.3.0`

### 원본 Q1~Q9 요청 표 (이력 보존)

모노레포 코드 점검 결과 SPEC과 어긋나거나 미구현 항목이 있어서 결정 필요했음. 위 답변 표에서 모두 처리됨.

| # | 우선순위 | 질문 | 옵션 | 영향 받는 R 항목 |
|---|---------|------|------|-----------------|
| Q1 | **P0** | `user-api-gateway`는 BFF인가 Spring Cloud Gateway인가? 코드는 BFF, SPEC §1.1은 SC Gateway | (a) BFF 보존 + SPEC 정정 / (b) SC Gateway 리라이트 / (c) 둘 다 (분리) | R-04 |
| Q2 | **P0** | `user/auth/identification`을 어떤 deployable로 합칠 것인가? 모노레포에 합쳐진 app이 없음 + AuthRestController @RestController 구현 부재 | (a) 한 서비스로 통합 / (b) auth 별도 분리 / (c) 다른 의도 | (신규) |
| Q3 | **P0** | JWT 검증 강제 활성화 시점? 현재 SecurityConfig는 `.permitAll()`로 검증 안 함 | (a) Phase 4 중 활성화 / (b) Phase 6 즈음 / (c) 영구 보류 | (신규) |
| Q4 | P1 | Kafka 직렬화 JSON 유지 vs Protobuf 마이그레이션? | (a) JSON 유지 / (b) Protobuf 동시 마이그레이션 (양쪽 ~5 파일씩) | R-13 |
| Q5 | P1 | Kafka 토픽명 모노레포 vs SPEC 표준? | (a) SPEC 표준(`order.pending` 등)으로 정렬 / (b) 모노레포 이름 유지 + SPEC 정정 | (신규) |
| Q6 | P1 | SPEC §0 payment/shipping 시뮬레이션 — 실제 구현 의도? `OrderLineItemStatus`에 해당 상태 부재 | (a) state machine 확장 / (b) INVENTORY_RESERVED=CONFIRMED로 단순화 / (c) 별도 작업 | (신규) |
| Q7 | P1 | `inventory-event` 모듈 — Event Sourcing vs Outbox? | (a) Outbox로 SPEC 정정 / (b) Event Sourcing 의도 (미완성) / (c) 다른 패턴 | (신규) |
| Q8 | P2 | Notification 서비스 설계 — channel 분기 / 멱등성 / DLQ | (Phase 0 후) | R-05 |
| Q9 | P2 | 모노레포 Kotlin 2.3.20 + Gradle 9.0.0 빌드 실패 인지 여부 | IntelliJ로만 빌드했는지 확인 | R-16 |

---

## 완료 이력

### 2026-05-12 — Kickoff (branch `bootstrap/spec-and-backlog`)
- `msa-argocd-manifest` 루트에 `TROICA_SPEC.md` v1.0 커밋
- 본 `BACKLOG.md` 신설
- 3개 기존 레포 상태 1차 점검 완료 (`msa-spring-boot` 모노레포 14모듈, `msa-provisioning` Terraform+Ansible 가동 중, `msa-argocd-manifest` 빈 상태)

### 2026-05-12 — Phase 1: 매니페스트 레포 골격 (branch `phase-1/manifest-skeleton`)

생성된 파일 트리:
```
msa-argocd-manifest/
├── README.md                                      (갱신)
├── TROICA_SPEC.md                                 (R-01, R-02 반영)
├── BACKLOG.md                                     (본 항목 추가)
├── bootstrap/
│   ├── root-app.yaml                              # kubectl apply 단일 진입점
│   └── README.md
├── projects/
│   ├── market-project.yaml                        # AppProject (sync-wave -10)
│   └── platform-project.yaml                      # AppProject (sync-wave -10)
├── applications/
│   ├── appset.yaml                                # ApplicationSet (_* exclude)
│   ├── charts/microservice/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── .helmignore
│   │   └── templates/{_helpers.tpl, deployment, service, hpa, serviceaccount, configmap, virtualservice}.yaml
│   └── values/
│       ├── README.md
│       └── _template/{values, values-dev, values-prod}.yaml
└── platform/
    ├── root.yaml                                  # App-of-Apps root (Phase 5 채워짐)
    └── README.md
```

검증 결과:
- `helm lint applications/charts/microservice` → 0 failure (1 info: icon 권장 — 무시)
- `helm template` 렌더링: default / `_template` dev / `_template` prod / worker enabled 모두 정상
- 모든 ArgoCD CRD YAML 구조 valid (apiVersion, kind, metadata.name 존재 — Python yaml.safe_load 기반 오프라인 검증)
- **클러스터-사이드 검증은 ArgoCD CRD가 설치된 클러스터에서 별도 수행 필요** (Phase 0/2 환경 준비 후)

SPEC §14 Phase 1 끝 체크리스트:
- [x] `helm template applications/charts/microservice -f applications/values/_template/values.yaml` 에러 없이 렌더링
- [ ] `kubectl --dry-run=client apply -f applications/appset.yaml` 통과 — 로컬에 클러스터 미연결로 미수행. ArgoCD 설치된 클러스터에서 수행 예정 (보류 항목 P1-V)

### 2026-05-12 — Phase 2: msa-common-libs 멀티모듈 + Protobuf (branch `phase-2/initial-setup` in `msa-common-libs`)

생성된 4-module Gradle 프로젝트:

```
msa-common-libs/
├── settings.gradle.kts (include 4 모듈)
├── build.gradle.kts (allprojects + subprojects: java-library + maven-publish + Spring Boot BOM)
├── gradle.properties (kotlin.code.style=official, kapt.use.k2=false)
├── gradle/wrapper/* (Gradle 8.10.2)
├── .github/workflows/{ci.yml, publish.yml}
├── common/        (build.gradle.kts + 11 .kt 파일 — 모노레포에서 복사)
├── client-redis/  (build.gradle.kts +  5 .kt 파일)
├── client-ses/    (build.gradle.kts +  3 .kt 파일)
└── events/
    ├── build.gradle.kts (com.google.protobuf 0.9.5 + Kotlin codegen)
    └── src/main/proto/troica/
        ├── order/v1/{order_pending, order_inventory_reserved, order_confirmed, order_cancelled}.proto
        └── notification/v1/notification_requested.proto
```

빌드 환경 (R-09에 정리):
- Kotlin 2.1.0 + Gradle 8.10.2 + JVM 21 (모노레포의 2.3.20/9.0.0은 `BuildUtilKt.clearJarCaches()` `NoClassDefFoundError`로 실패)
- Spring Boot BOM 3.3.0을 `io.spring.dependency-management`로만 import (org.springframework.boot 플러그인 미적용 — 라이브러리니까)

스키마 결정 (R-10에 정리):
- **Protobuf 단독.** Avro 미사용. PoC 단계 schema registry 없이 common-libs:events JAR로 클래스 공유.
- 호환: 필드번호 불변 + enum `*_UNSPECIFIED` fallback + breaking change 시 major bump.

검증 결과:
- `./gradlew build` SUCCESS (4 모듈 컴파일 + Protobuf codegen + jar/sources-jar/javadoc-jar)
- `./gradlew publishToMavenLocal` SUCCESS → `~/.m2/repository/com/troica/msa/{common,client-redis,client-ses,events}/0.1.0-SNAPSHOT/` 에 12개 artifact 확인
- ⚠ GitHub Packages publish는 `v0.1.0` 태그 푸시 시점에 실제 동작 검증 예정 (Phase 2의 마지막 단계 — 사용자 수행)

### 2026-05-12 — Phase 2 후속: gradlew 100755 픽스 (branch `phase-2/initial-setup`, 추가 커밋 `fd692b5`)

Phase 2 PR의 첫 CI 실행이 `./gradlew: Permission denied (exit 126)`로 실패. Windows에서 staged된 gradlew가 git index상 `100644`였기 때문. `git update-index --chmod=+x gradlew`로 `100755`로 변경 후 push → CI 통과. **이후 모든 신규 Gradle 레포는 처음부터 100755로 staging** (Phase 3 적용 완료).

### 2026-05-12 — Phase 3 (앱): msa-order-service polyrepo + worker 신규 (branch `phase-3/initial-setup`)

생성된 2-module Gradle 프로젝트:

```
msa-order-service/
├── settings.gradle.kts (include order, order-service)
├── build.gradle.kts (root, repositories: mavenLocal + GitHub Packages)
├── gradle.properties / gradle/wrapper/* (Gradle 8.10.2)
├── gradlew (mode 100755 from start)
├── .github/workflows/ci.yml (SPEC §7)
├── .gitignore, .gitattributes, .dockerignore, Dockerfile (SPEC §8)
├── order/             (build.gradle.kts + 25 .kt 파일 from monorepo)
│   └── src/main/kotlin/dev/ktcloud/black/order/{common,order,outbox}/
└── order-service/
    ├── build.gradle.kts
    ├── src/main/proto/order.proto (gRPC 정의 — monorepo와 동일)
    ├── src/main/resources/application{.yaml,-dev,-prod,-worker}.yaml
    └── src/main/kotlin/dev/ktcloud/black/order/service/
        ├── OrderServiceApplication.kt    (@EnableScheduling 추가)
        ├── OrderGrpcControllerAdapter.kt (monorepo)
        ├── adapter/web/inbound/OrderGrpcController.kt (monorepo)
        └── worker/OrderInventoryRequestOutboxPoller.kt  ← NEW (R-07 해소)
```

R-07 해소 — worker 분기 구현 디테일:
- 모노레포에 이미 `OrderInventoryRequestOutboxCommandService.processAll()`이 fetch unprocessed → Kafka publish → mark PUBLISHED 로직 완성. **유일한 결손은 주기적 호출자**였음.
- 추가한 빈: `OrderInventoryRequestOutboxPoller` (`@Profile("worker")` + `@Scheduled(fixedDelayString = "${order.worker.outbox.poll-interval-ms:5000}", initialDelayString = "${order.worker.outbox.initial-delay-ms:10000}")`) — 1 파일, 30라인.
- `application-worker.yaml`: `spring.main.web-application-type=none` + `grpc.server.port=-1` → worker pod에는 노출 포트 0.
- `@EnableScheduling`은 무조건 켜되 `@Scheduled` 빈이 `@Profile("worker")` 가드 → api Deployment에선 스케줄러 작업 자체가 등록되지 않음.

R-06 검증 완료:
- 모노레포 Spring Boot 3.3.0 확인
- bootJar `MANIFEST.MF`의 `Main-Class: org.springframework.boot.loader.launch.JarLauncher` 확인 → SPEC §8 Dockerfile 그대로 사용 가능

검증 결과:
- `./gradlew build -x test` SUCCESS (1m 24s) — kapt(QueryDSL) + Protobuf/gRPC codegen + bootJar 정상
- bootJar MANIFEST.MF + BOOT-INF/layers.idx 검증 — layered jar OK
- Helm 차트 `helm template` (dev + prod): api Deployment + worker Deployment + HPA(prod만) 정상 생성
- ⚠ Docker build는 본 환경에 docker 미설치로 보류 (R-12). CI에서 검증 예정
- ⚠ CI는 common-libs가 GitHub Packages에 publish된 후에만 통과 (R-11에 의존)

### 2026-05-12 — Tech stack audit + Spring Boot 3.3.0 → 3.5.13 upgrade

기술 스택 사진 (`05 — 기술 스택 / 버전 및 라이브러리 검토`) 표준과 코드 베이스 정합성 감사. 사진은 **Spring Boot 3.5.x**를 현행 표준으로 명시했으나 Phase 2/3에서 모노레포 카피로 인해 **3.3.0**이 들어갔음 — 발견 + 수정.

**감사 결과 (CVE/EOL)**
- Spring Boot 3.3.x: OSS support 2025-06-30 EOL ✕ (≒1년 EOL 상태로 사용 중이었음)
- Spring Boot 3.3.0 영향 CVE:
  - **CVE-2025-22235** (HIGH, CVSS 7.3) — `EndpointRequest.to()` actuator matcher 결함. 3.3.0-3.3.10 영향, 3.3.11에서 fix
  - **CVE-2025-24813** — 번들 Tomcat 10.1.24의 file-based session 역직렬화
  - **CVE-2025-66614 / CVE-2026-24733** — Tomcat 10.1.50 미만 영향
- 그 외 점검: Java 21 ✅ / Redisson 3.27.2는 CVE-2023-42809 fix 포함(3.22+)이지만 ~2년 구버전 / net.devh:grpc-server-spring-boot-starter 3.1.0 SB 3.2.4 컴파일 (호환 검증 필요) / Trivy v0.36.0 ✅

**선택지**: Spring Cloud Gateway WebFlux 호환성 검증 — 팀장님이 "맞는 게 없다"고 한 부분 확인:
- **Spring Cloud 2025.0.2 (Northfields, 2026-04-02 release)** = Spring Boot 3.5.13 공식 테스트 짝. **존재함.**
- Spring Cloud 2025.1.0 (Oakwood)은 **Spring Boot 4.x 전용** — Phase 4 시점에 잘못 보면 함정
- Spring Cloud 2024.0.x = SB 3.4.x까지만 — 3.5에 못 씀

**적용 (커밋)**
- `msa-common-libs` (branch `upgrade/spring-boot-3.5`, commit `63d2db1`): SB BOM 3.3.0 → 3.5.13. version 0.1.0-SNAPSHOT → 0.2.0-SNAPSHOT
- `msa-order-service` (branch `upgrade/spring-boot-3.5`, commit `a761659`): SB plugin + BOM 3.3.0 → 3.5.13, common-libs deps 0.1.0 → 0.2.0-SNAPSHOT

**Kotlin/Gradle 동시 upgrade 시도 → 후퇴 (R-16)**
- 시도: Kotlin 2.1.0 → **2.3.21** + Gradle 8.10.2 → **9.5.0**
- 실패: `BuildUtilKt.clearJarCaches()` `NoClassDefFoundError` (`ClasspathEntrySnapshotter$Settings` 누락) — Kotlin 2.3.x Build Tools API JAR 내부 클래스 참조 깨짐. JetBrains upstream 버그
- 워크어라운드 시도 (모두 실패): `kotlin.compiler.runViaBuildToolsApi=false`, `kotlin.incremental.useClasspathSnapshot=false`, `kotlin.incremental=false`
- **결정**: SB 3.5.13만 살리고 toolchain은 Kotlin 2.1.0 + Gradle 8.10.2 유지. JetBrains가 수정하면 재시도

**검증 결과**
- common-libs `./gradlew build` SUCCESS, `publishToMavenLocal` → `~/.m2/.../com/troica/msa/*/0.2.0-SNAPSHOT/` 12 artifact
- order-service `./gradlew build -x test` SUCCESS (1m 35s)
- bootJar MANIFEST: `Spring-Boot-Version: 3.5.13`, `Main-Class: org.springframework.boot.loader.launch.JarLauncher` 그대로 — Dockerfile 변경 불필요
- net.devh:grpc-server-spring-boot-starter:3.1.0 compile + link OK against SB 3.5.13 — 런타임 smoke test는 클러스터에서

**후속 필요 (선결)**
- 사용자: msa-common-libs 측 PR 머지 후 `git tag v0.2.0 && git push origin v0.2.0` → GH Packages publish
- 그 후 별도 PR: msa-order-service에서 common-libs 의존 `0.2.0-SNAPSHOT` → `0.2.0`

### 2026-05-12 — Phase 3 (매니페스트): order-service values 추가 (branch `phase-3/order-service-values`)

`applications/values/order-service/` 디렉토리 신설:
- `values.yaml` — 공통값 (image repo, port 8002, secretRef/configMapRef 이름, istio.enabled=true, worker section 기본)
- `values-dev.yaml` — replica 1, profile=dev, worker.enabled=true (profile dev,worker)
- `values-prod.yaml` — replica 3, profile=prod, HPA 3-10, worker.enabled=true (profile prod,worker, replica 2)

검증: `helm template ... -f values.yaml -f values-{dev,prod}.yaml` → ServiceAccount + Service + api Deployment + worker Deployment (+ prod HPA) 모두 정상.

### 2026-05-12 — Phase 4 후속: 5개 서비스 (branch별)

- `msa-product-service phase-4/initial-setup` — 단순 추출
- `msa-order-service phase-4/topics-and-state-machine` — Q5 토픽명 + Q6 state machine 확장 + OrderStatusOutbox + admin endpoint
- `msa-inventory-service phase-4/initial-setup` — Q5 + Q7 Event Sourcing + Spring Batch worker + JitPack client-redis
- `msa-user-service phase-4/initial-setup` — dual delivery (user lib publish + user-service app). user 라이브러리 v0.1.0 publish 완료 (feature branch tip 기준)
- `msa-auth-service phase-4/initial-setup` (신규 레포) — auth lib + auth-service app, gRPC `AuthService.{SignUp, SignIn, CheckValidity}`
- `msa-api-gateway phase-4/initial-setup` (단일 모듈) — Spring Cloud 2025.0.2 Northfields + BFF + Gateway routes 혼합 (Q1 (c))
- 매니페스트 PR 6개 동행 — 4개 머지 완료, auth-service / api-gateway 미머지

### 2026-05-12 — Phase 0: AWS OIDC + ECR + VPC Endpoint + kubelet credential provider

`msa-provisioning` 2개 stacked PR:

**PR 1 `phase-0/aws-ecr-oidc`** (commit `4478bf8`):
- `terraform/variables.tf` — region, github_org, msa_services 변수 + `aws_caller_identity` data source
- `terraform/oidc.tf` — GitHub OIDC IdP + `troica-gha-ecr-push` IAM role (assume 조건: msa-* repo main 브랜치 한정)
- `terraform/ecr.tf` — KMS key + 6 ECR repos (D6 후 notification 제외) + lifecycle (최근 30 이미지)
- `terraform/vpc-endpoint.tf` — SG + ECR Interface × 2 + S3 Gateway endpoint
- terraform fmt + validate 통과

**PR 2 `phase-0/kubelet-ecr-credential-provider`** (stacked on PR 1, commit `0d5e881`):
- `terraform/iam.tf` — node role에 `AmazonEC2ContainerRegistryReadOnly` attach
- `terraform/inventory.tftpl` + `terraform/ansible.tf` — `aws_account_id`, `aws_region` ansible inventory 주입
- `ansible/configuration/credential-provider-config.yaml.j2` — kubelet CredentialProviderConfig 템플릿
- `ansible/ecr-credential-provider-setup.yaml` — 신규 playbook (다운로드 + 배포 + drop-in + kubelet 재시작)
- `ansible/main.yaml` — argocd-setup 다음에 import

비용: KMS $1 + ECR Interface × 2 ≈ $14 = **약 $15-25/월**.

사용자 액션 (apply 절차):
1. PR 1 머지 → `cd terraform && terraform apply`
2. terraform output 확인 → GitHub Org Secrets `AWS_ACCOUNT_ID`, `MANIFEST_PAT` 등록
3. GitHub Org Variable `AWS_DEPLOYMENTS_ENABLED=true` 등록 → **R-19 해결, 6 서비스 CI push-gated job 일괄 활성**
4. PR 2 머지 → `terraform apply` (iam attach 반영) → `ansible-playbook -i ../ansible/inventory.ini ../ansible/main.yaml`
5. (matter of choice) `applications/values/<service>/values.yaml`의 image.repository placeholder ACCOUNT_ID를 실제 값으로 일괄 update (PR 1 output의 ecr_registry_url 사용)

---

## SPEC 변경 이력

| 일자 | 위치 | 변경 | 사유 |
|------|------|------|------|
| 2026-05-12 | §2 디렉토리 트리 | `order-service/values-worker.yaml` 라인 제거 + worker 분기는 flag로 한다는 주석 추가 | R-01 해결: dead file 제거 |
| 2026-05-12 | §6 multi-source 예시 | `source:` + `sources:` 중복을 `sources:` 단독으로 정리 | R-02 해결: ArgoCD validation 통과 |
| 2026-05-12 | §0 결정 사항 표 | "이벤트 스키마 = Protobuf", "Service mesh/Ingress = Istio" 행 추가 | Phase 2 결정: R-10, R-03 명시화 |
| 2026-05-12 | §1.2 공용 라이브러리 | 멀티모듈 4개 서브모듈 명시 (Avro/Protobuf → Protobuf 확정) | Phase 2 멀티모듈 구조 확정 |
| 2026-05-12 | §9.3 페이로드 스키마 | "Protobuf 또는 Avro" → "Protobuf only + no registry for PoC" 전면 재작성, 토픽↔proto 매핑 표 + 호환성 규칙 추가 | Phase 2 R-10 결정 |
| 2026-05-12 | §12 msa-common-libs | 단일모듈 예시 → 멀티모듈 publish 패턴 + 빌드환경 §12.0.1 신설 | Phase 2 실제 구현 반영 |
| 2026-05-12 | §13.2 Phase 2 작업 순서 | 단계 5개 → 9개로 확장 (멀티모듈, Protobuf codegen, Kotlin 2.1/Gradle 8.10 명시) | 실제 진행 결과 반영 |
| 2026-05-12 | §13.3 Phase 3 작업 순서 | 단계 11개 → 17개로 확장. inventory-event 이전 항목을 제거 (해당 모듈은 inventory 도메인 — Phase 4 inventory-service로). worker 신규 구현 명시. gradlew 100755 항목 추가 | Phase 3 실제 진행 + R-07 해소 + Phase 2 교훈 반영 |
| 2026-05-12 | §7 CI workflow Trivy 버전 | `aquasecurity/trivy-action@0.24.0` → `@v0.36.0` + 공급망 사고 주석 | R-14: 0.24.0 미존재 + 0.0.1~0.34.2 태그는 2026-03 공급망 사고로 compromised. post-incident 태그 핀 |
| 2026-05-12 | §0 결정표 + §12.0.1 | Spring Boot `3.3.0` → `3.5.13` (BOM). 빌드 toolchain / Spring Cloud 행 신설 (SC 2025.0.2 Northfields 명시) | 사진 표준 정렬 + Spring Boot 3.3 EOL + CVE-2025-22235 + Phase 4 api-gateway 호환성 |
| 2026-05-12 | §13.3 step 7 | "SB 3.3.0 검증됨" → "SB 3.5.13 검증됨" (JarLauncher 경로 동일) | upgrade 결과 반영 |
| 2026-05-12 | §7 CI workflow + §7.1 Secrets 표 | 7개 push-gated `if`에 `&& vars.AWS_DEPLOYMENTS_ENABLED == 'true'` 추가. §7.1 표를 Secret/Variable 분리 + 신규 Variable 행 추가 | R-19: Phase 0 미완 상태에서 main CI 영구 빨간 상태 회피. Phase 0 후 1-flag 활성화 |

---

## 보류 / 검토 필요

| # | 항목 | 위치 | 사안 | 결정 시점 / 상태 |
|---|------|------|------|----------|
| R-01 | ~~`values-worker.yaml` 별도 파일 미사용~~ | SPEC §2 | ~~dead file~~ | **해결 (2026-05-12, Phase 1)** |
| R-02 | ~~App-of-Apps multi-source 스니펫 오류~~ | SPEC §6 | ~~`source:`+`sources:` 중복~~ | **해결 (2026-05-12, Phase 1)** |
| R-03 | Traefik → Istio 교체 | `msa-provisioning` | Istio가 최종 mesh로 확정 (2026-05-12 사용자 컨펌). Phase 5에서 Traefik 제거 + Istio platform Application 추가 | Phase 5 |
| R-04 | `user-api-gateway` 명칭 불일치 | 모노레포 vs SPEC §1.1 | 신규 레포명은 `msa-api-gateway`, 내부 module명 결정 보류 | Phase 4 |
| R-05 | `notification-service` 그린필드 | SPEC §9 | 코드 0줄, 완전 신규 | Phase 4 |
| R-06 | Spring Boot 버전 + JarLauncher 경로 | SPEC §8 | Dockerfile 작성 전 jar manifest 확인 | Phase 3 |
| R-07 | `order-worker` profile 분기 코드 검증 | SPEC §1.1 | 모노레포 order-service에 worker profile 분기 코드 미확인 | Phase 3 |
| R-08 | `msa-spring-boot` 아카이브 시점 | SPEC §1.3 | 모든 서비스 이전 완료 후 read-only | Phase 6 |
| R-09 | Kotlin/Gradle 버전 충돌 (모노레포 vs common-libs) | Phase 2 | 모노레포는 Kotlin 2.3.20 + Gradle 9.0.0 선언이지만 CLI 빌드 시 `BuildUtilKt.clearJarCaches()` 미해결로 실패 → common-libs는 Kotlin 2.1.0 + Gradle 8.10.2로 진행. **소비자(서비스 레포)도 안정 조합 사용 권장** | Phase 3+ 모든 서비스 레포는 Kotlin 2.1.0 + Gradle 8.10.2 + JVM 21로 통일 |
| R-10 | ~~이벤트 스키마 Protobuf vs Avro~~ | SPEC §9.3 | ~~SPEC v1.0에 "Avro 또는 Protobuf" 양립~~ | **해결 (2026-05-12, Phase 2)** Protobuf 단독. |
| R-06 | ~~Spring Boot 버전 + JarLauncher 경로~~ | SPEC §8 | ~~JarLauncher 경로 확인~~ | **해결 (2026-05-12, Phase 3)** SB 3.3.0, `org.springframework.boot.loader.launch.JarLauncher` 확인 — SPEC §8 그대로 사용 |
| R-07 | ~~`order-worker` profile 분기 코드 검증~~ | SPEC §1.1 | ~~worker profile 미존재~~ | **해결 (2026-05-12, Phase 3)** `@Profile("worker")` + `@Scheduled` 빈 신규 1개로 해소. 기존 `processAll()` 재사용 |
| R-11 | ~~GitHub Packages publish 실 검증 (common-libs)~~ | Phase 2 | ~~v0.1.0 태그 푸시 + publish 워크플로우 동작 검증~~ | **해결 (2026-05-12, 사용자 수행)** v0.1.0 publish 성공. 4 모듈 GH Packages에 게시. Phase 3 의존성을 `0.1.0`(stable)로 bump 완료 (`c1facfd`) |
| R-12 | Phase 3 로컬 Docker build 미검증 | Phase 3 | 본 환경에 docker 미설치. Dockerfile은 SPEC §8 모델 + bootJar 매니페스트 검증으로 간접 신뢰. 실제 검증은 CI의 `docker build` step (Phase 0 후) 또는 사용자 로컬 docker 환경 | CI 첫 docker build step 실행 시 |
| R-13 | Kafka 직렬화 JSON → Protobuf 마이그레이션 | Phase 3+ | Phase 3 스코프 폭주 방지를 위해 기존 JSON 유지. order-service는 common-libs:events 의존만 받음. wire 마이그레이션은 inventory-service와 동시 진행이 필요 (둘 다 바꿔야 양립 가능) | Phase 4 inventory-service 작업과 묶어서 |
| R-14 | ~~SPEC §7 `aquasecurity/trivy-action@0.24.0` 미존재 + 보안 사고 범위~~ | SPEC §7, 모든 서비스 CI | ~~버전 미존재로 "Prepare all required actions"에서 실패 (`if` 조건 평가 전이라 conditional step도 영향). 게다가 2026-03-19 trivy-action 공급망 사고로 0.0.1~0.34.2 태그는 compromised~~ | **해결 (2026-05-12)** SPEC §7 + msa-order-service ci.yml을 `v0.36.0` (post-incident, v-prefix)로 핀. 추가 보안 강화 = SHA pin + Dependabot은 backlog 보류 |
| R-15 | GitHub Actions 핀 정책 (SHA + Dependabot) | 보안 강화 | 현재 mutable tag 핀. 공급망 공격에 취약. 모든 actions를 commit SHA로 핀 + Dependabot/Renovate 도입 권장 | Phase 6 또는 별도 보안 트랙 |
| R-16 | Kotlin 2.3.x + Gradle 9.x `BuildUtilKt.clearJarCaches()` 버그 | toolchain | Kotlin 2.3.20/2.3.21 + Gradle 9.0/9.5 모두 `ClasspathEntrySnapshotter$Settings` `NoClassDefFoundError`. 워크어라운드 3종(`runViaBuildToolsApi=false`, `useClasspathSnapshot=false`, `incremental=false`) 전부 무효. JetBrains upstream 이슈로 보임 | Kotlin 2.4.x 또는 2.3.22+ release 후 재시도. 그 전에는 2.1.0 + 8.10.2 고정 |
| R-17 | common-libs v0.2.0 publish 후 order-service deps 재bump | upgrade 후속 | order-service는 현재 `com.troica.msa:*:0.2.0-SNAPSHOT`. common-libs v0.2.0 태그 push로 GH Packages publish 후 별도 PR로 `:0.2.0`으로 변경 (Phase 3 v0.1.0 때와 동일 패턴) | 사용자 v0.2.0 태그 push 후 |
| R-18 | Spring Boot 3.5 OSS EOL 2026-06-30 | 일정 | 현재 3.5.13 사용. 6월 30일 이후 보안 패치 OSS 미제공. Phase 6 마무리 후 3.6.x/4.0 마이그레이션 검토 (Spring Cloud 2025.1.x = Boot 4 라인) | Phase 6 종료 시점 또는 EOL 2주 전 |
| R-19 | CI push-gated step의 AWS_DEPLOYMENTS_ENABLED 게이트 | Phase 3 후속 | order-service 머지 후 main CI가 OIDC AssumeRole 실패로 영구 빨간 상태 → 팀이 "원래 빨갛다"에 익숙해지는 위험. `vars.AWS_DEPLOYMENTS_ENABLED == 'true'` 가드 추가로 변수 미설정 시 push-gated step skip → main CI 녹색. Phase 0 완료 후 Org 레벨 변수를 `true`로 set하면 자동 활성 | **Phase 0 PR 머지 + terraform apply + Org Variable 등록 시 자동 해결 (Phase 0 PR 산출물에 절차 명시)** |
| R-26 | 매니페스트 values의 image.repository placeholder ACCOUNT_ID | Phase 0 후속 | 모든 `applications/values/<service>/values.yaml`의 `image.repository`가 `000000000000.dkr.ecr.ap-northeast-2.amazonaws.com/msa/...` placeholder. Phase 0 terraform output의 실제 account_id로 일괄 치환 필요 | Phase 0 apply 후 별도 매니페스트 PR로 sed 일괄 변경 |
| R-20 | JitPack client-redis 운영 모드 | Phase 4 | 팀장님 별도 GitHub 레포 + JitPack publish 운영. 우리 통제 밖 — JitPack 빌드 실패 가능성, 버전 lifecyle 가시성 낮음. 모니터링 필요 시 dependabot/renovate로 버전 변경 알림만 | 운영 모니터링 |
| R-21 | Q6 order state machine 확장 구현 | Phase 4 (#3) | `OrderLineItemStatus` enum 확장 + `checkTransitive` 갱신 + 새 Outbox 테이블(or 기존 일반화) + `order.confirmed`/`order.cancelled` 이벤트 발행자 | Phase 4 진행 중 |
| R-22 | Q7 inventory Event Sourcing + Spring Batch worker | Phase 4 (#4) | `inventory-event`를 append-only 스토어로 변환 (processed flag 제거 또는 일관 유지). `@EnableBatchProcessing` + Job/Step. worker 프로파일 + 별도 K8s Deployment. spring-boot-starter-batch 의존성 추가 | Phase 4 진행 중 |
| R-23 | api-gateway BFF + SC Gateway 혼합 — 어떤 path가 어디로? | Phase 4 (#7) | 라우팅 매트릭스 필요: 어떤 path는 BFF REST controller로, 어떤 path는 SC Gateway route로. 현재 모노레포 코드는 product/order/inventory를 BFF로 처리. 신규 추가될 SC Gateway routes 대상 미정 | Phase 4 #7 진행 직전 결정 |
| R-24 | identification 모듈 제거에 따른 user 도메인 영향 | Phase 4 (#5) | 모노레포에서 identification 폐기됨. user/auth 코드에서 identification 의존 흔적 제거 확인. 이메일 검증 기능은? | user-service / auth-service 작업 시 확인 |
| R-25 | `auth-service`의 JWT secret 외부화 | Phase 4 / 5 | 모노레포 application.yaml에 평문 secret(`v7S6A9yB2E5H8KcNfUjXnZr4u7x!A%D*G-`). PoC라도 K8s Secret + ExternalSecrets로 분리 필요. Phase 0 후 정식 secret 운영 | Phase 0 후 |
| P1-V | Phase 1 클러스터-사이드 검증 미수행 | 로컬 환경 | 로컬에 ArgoCD CRD 설치된 클러스터 미연결 → kubectl dry-run 불가. 오프라인 YAML 구조 검증만 통과 | Phase 0/2 인프라 준비 후 실 클러스터에서 `kubectl apply -f bootstrap/root-app.yaml --dry-run=server` 수행 |
| R-27 | CI 파이프 최적화 | Phase 5 | Phase 0 첫 활성화 시 6개 서비스에서 관찰된 비효율 항목들. 본 PoC는 작동하지만 운영 효율 개선 여지. (a) Gradle 중복 빌드 제거 — 호스트 `./gradlew build` + Docker 내부 bootJar 중복 → Docker 단일 실행으로 통합 시 -1.5분/빌드. (b) Docker BuildKit cache mount for gradle deps (`--mount=type=cache,target=/root/.gradle`) → -30~60s/빌드. (c) Trivy DB cache key 갱신 — 현재 날짜 단위 → 주 단위로 늘려 매일 첫 빌드의 ~1분 단축. (d) Reusable workflow / composite action — 6개 ci.yml 거의 동일, DRY 위반. 단일 워크플로우 + 서비스별 env만 분리. (e) `paths-ignore: ['**.md', 'docs/**', 'LICENSE']` → 도큐먼트만 변경된 PR/push에서 빌드 skip. (f) ECR login retry wrapper — 5개 서비스 동시 머지 시 ECR `GetAuthorizationToken` rate limit, 재시도 자동화. (g) Dependabot/Renovate 활성화 — CVE를 사용자가 직접 발견하기보다 자동 PR로 사전 감지. patch 버전 auto-merge 가능 | Phase 5 — 클러스터 셋업 + ArgoCD sync 완료 후 |
| R-28 | netty 4.2.x BOM 전환 + CVE-2026-42577 해제 | 외부 의존성 | api-gateway의 `.trivyignore`에 CVE-2026-42577 (netty-transport-native-epoll, RST on half-closed TCP DoS) accepted. fix는 netty 4.2.13.Final에만 존재 (4.1.x line은 fix 없음). Spring Boot 3.5.x BOM이 netty 4.2.x로 갱신되거나 reactor-netty가 4.2.x 호환 발표 시 즉시 `.trivyignore` 항목 제거 + netty 명시 override 제거 | Spring Boot 3.5.15+ 또는 reactor-netty 1.3.x 출시 모니터링 |
| R-29 | cloud-provider-aws ecr-credential-provider 바이너리 배포 패턴 | Phase 0 후속 (해결) | cloud-provider-aws v1.30.x release에 binary asset 게시 안 됨 (assets=0). 본 PoC는 Go 소스에서 직접 빌드 후 ansible local→remote copy 패턴 채택. msa-provisioning `fix/ansible-ecr-credential-provider-integrated` PR로 정식화. 추후 EKS AMI 사용 또는 자체 baked AMI 도입 시 본 절차 폐기 가능 | 해결 완료 — playbook의 vars 주석에 빌드 절차 명시 |
| R-30 | kubelet KUBELET_EXTRA_ARGS drop-in 미평가 이슈 | Phase 0 후속 (해결) | Amazon Linux 2023 + kubeadm 1.30 환경에서 `/etc/systemd/system/kubelet.service.d/30-ecr-credential-provider.conf`의 `Environment="KUBELET_EXTRA_ARGS=..."`가 systemd에 의해 평가되지 않음 (kubelet `ps aux`에 인자 미반영). 원인 미규명. 우회: `/var/lib/kubelet/kubeadm-flags.env`의 `KUBELET_KUBEADM_ARGS`에 ECR args 직접 lineinfile로 통합. drop-in은 다른 환경 호환 위해 유지. msa-provisioning `fix/ansible-ecr-credential-provider-integrated`에 통합 | 해결 완료. 근본 원인 규명은 Phase 6 발표 후 |
| R-31 | ArgoCD `directory.include` brace expansion `{a,b,c}` 미동작 | 매니페스트 | `bootstrap/root-app.yaml`의 `directory.include: '{projects/market-project.yaml,projects/platform-project.yaml,platform/root.yaml,applications/appset.yaml}'`이 ArgoCD에서 4개 파일 중 0개 매칭. Demo 시 manual `kubectl apply`로 우회. 정공법: (a) `directory.recurse: true` + path 구조 정리, 또는 (b) bootstrap을 4개 별도 Application으로 분리 (app-of-apps 패턴), 또는 (c) `include`에 glob `*.yaml`만 명시 후 해당 디렉토리에 다른 yaml 안 두도록 | 차기 작업 세션 첫 작업 |
| R-32 | `ansible/argocd-setup.yaml`의 root-app repo URL이 팀장님 개인 레포 | Phase 0 후속 | 현재 `kanei0415/ktcloud-k8s-argocd-manifest` 가리킴 (팀장님 PoC 레포). 클러스터 부트스트랩 시 즉시 우리 팀 매니페스트 미동기화. Demo에서는 manual `kubectl delete root-app` + 우리 root-app 재배포로 우회. 정공법: ansible playbook의 root-app yaml 정의 (또는 `argocd app create` 명령)를 `KTCloud-CloudNative-Troica-Team/msa-argocd-manifest`로 변경 + Setup→. 경로로 변경. R-31과 함께 진행 | 차기 작업 세션 첫 작업 |
| R-33 | 6개 서비스 × 2 env Secret + ConfigMap 외부 의존성 | Phase 5 (platform) | 매니페스트 chart가 `{service}-secrets`, `{service}-config`를 envFrom으로 참조. PoC demo는 placeholder Secret/ConfigMap으로 pod startup만 충족. 실제 운영 시 ExternalSecrets Operator + AWS Secrets Manager (또는 SealedSecrets) + 환경별 실값 주입 필요. Phase 5 platform 배포(R-35)와 함께 묶어서 진행 | Phase 5 |
| R-34 | `destroy-temp.sh` Windows 호환 + git autocrlf 이슈 | Phase 0 후속 (부분 해결) | Windows에서 git core.autocrlf 기본값 때문에 `.sh` 파일이 CRLF로 저장 → bash 실행 시 `$'\r': command not found` 에러. msa-provisioning `chore/destroy-temp-powershell` PR로 PowerShell 네이티브 `destroy-temp.ps1` 추가 (머지 완료). 추가 권장: `.gitattributes`에 `*.sh text eol=lf` 강제로 Windows에서도 LF 유지 → Linux/Mac 협업자와 Windows 사용자 모두 안전 | `.gitattributes` 추가 PR 별도 진행 |
| R-35 | Phase 5 platform 배포 | Phase 5 | (a) PostgreSQL: 6개 서비스용. Bitnami helm chart 또는 CloudNativePG operator. namespace 분리 또는 multi-database. (b) Redis: auth(token cache) + inventory(snapshot cache)용. Bitnami helm chart 또는 Redis Operator. (c) Strimzi Kafka Operator + KafkaTopic CRD × 4 (order.pending/inventory-reserved/confirmed/cancelled). (d) Istio Gateway + VirtualService — Traefik 제거. (e) AlertManager + Slack receiver — Grafana alerts. (f) Prometheus + Grafana + Loki 관측성 스택. (g) ExternalSecrets Operator (R-33과 묶음) | Phase 5 본 작업 |
| R-36 | ansible playbook + README 일본어 → 한국어 번역 | 코드 품질 | msa-provisioning에 일본어 14개 파일: `ansible/{k8s-pre-setup,k8s-pkg-install,containerd-setup,master-init,master-cni-setup,master-python-setup,join-master,join-worker,helm-setup,nlb-setup,argocd-setup,traefik-setup,k8s-clear}.yaml` + `README.md`. 팀장님 작성 코드의 `name:` 필드 + 주석이 일본어라 실행 출력 + 코드 리뷰 가독성 저하. 한 PR로 전체 일괄 번역 (의미 보존 + 줄 단위 1:1 매핑) | 사용자 요청, 별도 PR로 |

---

## 누적 결정 사항 (SPEC §0 외 추가)

| 일자 | 결정 | 근거 |
|------|------|------|
| 2026-05-12 | Istio가 최종 ingress/service mesh. Traefik은 Phase 5에서 제거 | 사용자 명시 |
| 2026-05-12 | `msa-spring-boot` 레포는 read-only로만 다룬다 (다른 작업자 충돌 회피) | 사용자 명시 |
| 2026-05-12 | Phase 진행 순서를 SPEC §13의 0 → 1 에서 **1 → 0 → 2 → ...** 로 조정 | Phase 0(Terraform apply)이 비가역이라 골격을 먼저 완성하고 컨펌 후 진행. 의존 깨지지 않음 |
| 2026-05-12 | ApplicationSet에 `_*` exclude 패턴 추가 — `_template` 디렉토리 도입 | 신규 서비스 추가 시 복사 시작점 + Phase 1 helm template 검증 대상. SPEC에는 미반영 (구현 디테일) |
| 2026-05-12 | `msa-common-libs`는 멀티모듈 (4개 서브모듈) | 소비자가 필요한 모듈만 의존 → ClassPath 슬림화. 사용자 명시 |
| 2026-05-12 | 이벤트 스키마 Protobuf 단독, schema registry 미도입 (PoC) | 모노레포 gRPC가 이미 Protobuf 사용. SemVer + common-libs:events JAR로 호환 관리 |
| 2026-05-12 | Kotlin 2.1.0 + Gradle 8.10.2 + JVM 21 (모든 서비스 레포에 동일 적용) | R-09: 모노레포 선언 버전(2.3.20/9.0.0)은 CLI에서 빌드 실패. 안정 조합으로 통일 |
| 2026-05-12 | 새 Gradle 레포는 처음부터 `gradlew` mode 100755로 staging | Phase 2 PR이 100644로 인해 Linux CI에서 Permission denied 실패한 교훈 |
| 2026-05-12 | `inventory-event` 모듈은 Phase 3 이전 대상 아님 — Phase 4 inventory-service로 | 패키지가 inventory 도메인 (`dev.ktcloud.black.inventory.event`). SPEC §13.3 step 2의 "등" 부분에서 제거 |
| 2026-05-12 | order-service의 Kafka 직렬화는 일단 기존 JSON 유지. Protobuf 마이그레이션은 R-13으로 분리 | Phase 3 스코프 폭주 방지. 직렬화 변경은 producer/consumer가 동시에 바꿔야 양립 가능 → inventory-service와 묶음 |
| 2026-05-12 | Spring Boot 3.5.13 표준 + Spring Cloud 2025.0.2 (Phase 4) | 사진 "기술 스택" 슬라이드 표준 + 3.3 EOL + CVE-2025-22235 (HIGH 7.3). Phase 4 api-gateway가 SC Gateway WebFlux 필요 — 2025.0.2가 SB 3.5.x 공식 짝 |
| 2026-05-12 | Kotlin/Gradle은 2.1.0 + 8.10.2 고정 유지 (Kotlin 2.3.x 시도 → 실패 후 후퇴) | R-16: JetBrains Build Tools API JAR 버그. SB 3.5.13는 Kotlin 2.1.0과 정상 작동 (SB 3.5의 BOM-managed Kotlin이 2.1.x 라인) |
