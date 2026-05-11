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

(없음)

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

### 2026-05-12 — Phase 3 (매니페스트): order-service values 추가 (branch `phase-3/order-service-values`)

`applications/values/order-service/` 디렉토리 신설:
- `values.yaml` — 공통값 (image repo, port 8002, secretRef/configMapRef 이름, istio.enabled=true, worker section 기본)
- `values-dev.yaml` — replica 1, profile=dev, worker.enabled=true (profile dev,worker)
- `values-prod.yaml` — replica 3, profile=prod, HPA 3-10, worker.enabled=true (profile prod,worker, replica 2)

검증: `helm template ... -f values.yaml -f values-{dev,prod}.yaml` → ServiceAccount + Service + api Deployment + worker Deployment (+ prod HPA) 모두 정상.

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
| P1-V | Phase 1 클러스터-사이드 검증 미수행 | 로컬 환경 | 로컬에 ArgoCD CRD 설치된 클러스터 미연결 → kubectl dry-run 불가. 오프라인 YAML 구조 검증만 통과 | Phase 0/2 인프라 준비 후 실 클러스터에서 `kubectl apply -f bootstrap/root-app.yaml --dry-run=server` 수행 |

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
