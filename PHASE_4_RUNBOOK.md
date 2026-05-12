# Phase 4 Runbook — 5개 서비스 폴리레포화

> **목적**: Phase 3(`msa-order-service`)에서 확정된 패턴을 5개 서비스에 일관 적용하기 위한 액션 레시피.
> SPEC([`TROICA_SPEC.md`](./TROICA_SPEC.md))은 원칙·청사진, 본 문서는 실행 절차.
> 결정 변경 시 SPEC 본문 갱신 후 본 문서도 동기화.

---

## 0. 사전조건 체크

- [x] Phase 3 완료 — `msa-order-service` main에 SB 3.5.13 + worker 빈 + AWS_DEPLOYMENTS_ENABLED 게이트 머지됨
- [x] `msa-common-libs:0.2.0` GitHub Packages 발행 완료
- [x] 신규 5개 빈 레포(LICENSE+README) 생성됨 (`msa-user-service`, `msa-product-service`, `msa-inventory-service`, `msa-notification-service`, `msa-api-gateway`)
- [ ] 팀장님 P0 질문 3개(Q1~Q3) 답변 받음 — BACKLOG의 "팀장님 보류 사항" 참조

> 팀장님 답변 (2026-05-12) 받은 후 매핑 갱신됨. 자세한 결정은 BACKLOG 참조.

---

## 1. 모노레포 → 폴리레포 매핑 (v2)

| 폴리레포 | 가져갈 모노레포 모듈 | deployable app? | 비고 |
|----------|---------------------|---------------|------|
| `msa-product-service`   | `product` + `product-service` | ✅ Phase 4 첫 PoC 완료 | — |
| `msa-order-service` (이미 폴리레포) | — | ✅ Phase 3 완료 | **Q5 토픽명 정렬 + Q6 state machine 확장** 후속 작업 |
| `msa-inventory-service` | `inventory` + `inventory-service` + `inventory-event` | ✅ `InventoryServiceApplication` 존재 | Redis(**JitPack**) + Kafka + **Q5 토픽명** + **Q7 Event Sourcing + Spring Batch worker** |
| `msa-user-service`      | `user/` (단독) | ❌ **신규 패키징 필요** | identification 폐기됨. `@SpringBootApplication` + REST/gRPC adapter 작성 |
| `msa-auth-service` (신규 레포 D1) | `auth/` + `auth-service/` | ✅ `AuthServiceApplication` 존재 | gRPC `AuthService.{SignUp, SignIn, CheckValidity}`. PostgreSQL `auth_db`:7007 + Redis :7006 |
| `msa-api-gateway`       | `user-api-gateway` | ✅ `UserApiGatewayApplication` 존재 | **Q1 (c) BFF + SC Gateway 혼합**. `JwtHeaderCheckFilter` 포함. Spring Cloud BOM 2025.0.2 |
| ~~`msa-notification-service`~~ | (폐기 D6 archive) | — | Q3 결정: 시스템 알림은 Prometheus → AlertManager → Slack 경로 |

---

## 2. 포트 / DB / 외부 의존성 매트릭스 (v2)

### 2.1 네트워크 포트

| 서비스 | HTTP | gRPC | DB | Redis | 비고 |
|--------|------|------|----|----|------|
| product-service   | 8001 | 9001 | 7001 (product_db) | - | |
| order-service     | 8002 | 9002 | 7002 (order_db)   | - | worker pod (Outbox poller) — 포트 없음 |
| inventory-service | 8003 | 9003 | 7004 (inventory_db) | 7005 (inventory) | **+worker pod (Spring Batch projection)** |
| user-service      | 8004 | 9004 | (별도 user_db 신규) | - | gRPC 노출 — auth-service가 `:user` 라이브러리로 사용 |
| auth-service      | 8005 | 9005 | 7007 (auth_db, 신규) | 7006 (auth-refresh-token, 신규) | 8005는 ~~notification~~ → auth로 재할당 |
| api-gateway       | 8100 | -    | - | - | 외부 진입점 |

### 2.2 외부 의존성

| 서비스 | PostgreSQL | Redis | Kafka |
|--------|-----------|-------|-------|
| product-service   | ✅ product_db | - | - |
| order-service     | ✅ order_db   | - | producer(`order.pending`, `order.confirmed`, `order.cancelled`) + consumer(`order.inventory-reserved`) |
| inventory-service | ✅ inventory_db | ✅ inventory-redis | consumer(`order.pending`) + producer(`order.inventory-reserved`) |
| user-service      | ✅ user_db    | - | - |
| auth-service      | ✅ auth_db    | ✅ auth-redis | - |
| api-gateway       | - | - | - |

> 모노레포 `container-compose.yaml`은 product/order/inventory + Kafka + inventory-redis만 정의. user/auth는 신규 추가 필요 (각 서비스 README에 로컬 docker-compose snippet 첨부 권장).

### 2.3 공용 라이브러리 의존성 (common-libs v0.3.0 기준)

| 서비스 | `:common` | `:events` | JitPack client-redis |
|--------|:---------:|:---------:|:--------------------:|
| product-service   | ✅ | - | - |
| order-service     | ✅ | (코드 양립용 보유) | - |
| inventory-service | ✅ | (코드 양립용 보유) | ✅ |
| user-service      | ✅ | - | - |
| auth-service      | ✅ | - | ✅ (refresh-token 캐시) |
| api-gateway       | (간접) | - | - |

> `client-ses`, `client-redis` (common-libs)는 v0.3.0에서 제거. JitPack은 `com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`.

---

## 3. 서비스 부트스트랩 레시피 (per service)

> Phase 3에서 검증된 패턴. 각 서비스마다 반복.
> 대문자 `<SERVICE>`, `<PORT_HTTP>`, `<PORT_GRPC>` 부분만 치환.

### 3.1 로컬 클론 + 브랜치

```bash
cd C:/Users/melan/ktcloud-troica
git -C <SERVICE> checkout -b phase-4/initial-setup
```

### 3.2 Gradle wrapper + 디렉토리 골격 복사 (모노레포에서 read-only)

```bash
SRC=/c/Users/melan/ktcloud-troica/msa-spring-boot
DST=/c/Users/melan/ktcloud-troica/<SERVICE>

mkdir -p "$DST/gradle/wrapper" "$DST/.github/workflows"
cp "$SRC/gradlew" "$SRC/gradlew.bat" "$DST/"
cp "$SRC/gradle/wrapper/gradle-wrapper.jar" "$DST/gradle/wrapper/"
```

`gradle-wrapper.properties`는 **Gradle 8.10.2**로 작성:
```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.10.2-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

### 3.3 코드 이전 (모노레포 → 폴리레포)

매핑 표(§1) 참조. 예: product-service의 경우
```bash
cp -R "$SRC/product/src" "$DST/product/"
cp -R "$SRC/product-service/src" "$DST/product-service/"
```

### 3.4 `settings.gradle.kts` (멀티모듈 패턴)

```kotlin
pluginManagement {
    repositories { mavenCentral(); gradlePluginPortal() }
}

plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "1.0.0"
}

dependencyResolutionManagement {
    repositories { mavenCentral() }
}

rootProject.name = "<service-name>"

include(
    "<domain-module>",      // e.g. "product"
    "<app-module>",         // e.g. "product-service"
)
```

### 3.5 `gradle.properties`

```properties
group=com.troica.msa
version=0.1.0-SNAPSHOT

kotlin.code.style=official
kapt.use.k2=false

org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
```

### 3.6 루트 `build.gradle.kts` (Spring Boot 앱용 — order-service와 동일 패턴)

`msa-order-service/build.gradle.kts` 그대로 복사 후 `mavenBom` 버전 확인:
- Kotlin plugin: `2.1.0`
- Spring Boot plugin: `3.5.13`
- Spring Boot BOM: `3.5.13`
- GitHub Packages repo URL: msa-common-libs

api-gateway만 추가 BOM 필요:
```kotlin
mavenBom("org.springframework.cloud:spring-cloud-dependencies:2025.0.2")
```

### 3.7 서브모듈 `build.gradle.kts`

도메인 모듈(예: `product/build.gradle.kts`)은 모노레포의 그것을 거의 그대로 복사하되, `project(":common")` 같은 monorepo 모듈 참조를 GitHub Packages 의존성으로 치환:

```kotlin
// 변경 전 (monorepo):
implementation(project(":common"))
implementation(project(":client-redis"))

// 변경 후 (polyrepo):
implementation("com.troica.msa:common:0.2.0")
implementation("com.troica.msa:client-redis:0.2.0")
```

### 3.8 `application.yaml` 프로파일 분리

모노레포의 `application.yaml`을 base로 두고, 환경변수 driven으로 수정:
- `${POSTGRES_HOST:localhost}`, `${KAFKA_BOOTSTRAP_SERVERS:localhost:7003}` 등
- `application-dev.yaml` (ddl-auto: update, DEBUG log)
- `application-prod.yaml` (ddl-auto: validate, INFO log)
- (해당 시) `application-worker.yaml` (web off, gRPC port=-1)

### 3.9 `Dockerfile` (SPEC §8 그대로 + EXPOSE 포트만 수정)

```dockerfile
EXPOSE <PORT_HTTP> <PORT_GRPC>     # gRPC 없으면 HTTP만
```

다른 라인은 order-service와 동일.

### 3.10 `.github/workflows/ci.yml` (SPEC §7 그대로 + env만 수정)

```yaml
env:
  AWS_REGION: ap-northeast-2
  SERVICE_NAME: <service-name>          # 예: product-service
  ECR_REPO: msa/<service-name>          # 예: msa/product-service
  MANIFEST_REPO: KTCloud-CloudNative-Troica-Team/msa-argocd-manifest
```

push-gated step에는 `&& vars.AWS_DEPLOYMENTS_ENABLED == 'true'` 가드 필수. 누락하면 Phase 0 미완 상태에서 main CI가 빨갛게 됨.

Trivy action은 **`@v0.36.0`** (Phase 3 R-14: 0.0.1~0.34.2 compromised).

### 3.11 `.gitignore`, `.gitattributes`, `.dockerignore`

order-service 동일. 복사.

### 3.12 **gradlew 100755 staging (Phase 2 교훈, 반드시!)**

```bash
git add gradlew
git update-index --chmod=+x gradlew
git ls-files --stage gradlew     # 100755 확인
```

### 3.13 로컬 빌드 검증

```bash
./gradlew build -x test --no-daemon
unzip -p <app-module>/build/libs/*.jar META-INF/MANIFEST.MF | head -3
# Spring-Boot-Version: 3.5.13, Main-Class: org.springframework.boot.loader.launch.JarLauncher 확인
```

### 3.14 커밋·푸시·PR

```bash
git add -A
git commit -m "feat(phase-4): bootstrap <service-name>"
git push -u origin phase-4/initial-setup
```

PR 본문에는 매핑 표 + 검증 결과 포함.

### 3.15 매니페스트 레포에 values 추가 (별도 PR)

`msa-argocd-manifest`에서 새 브랜치:
```
applications/values/<service-name>/
├── values.yaml          # nameOverride, image.repository, service.port, envFrom 등
├── values-dev.yaml      # tag=PLACEHOLDER, replicaCount, springProfilesActive=dev
└── values-prod.yaml     # tag=PLACEHOLDER, hpa, springProfilesActive=prod
```

ApplicationSet이 자동으로 `<service-name>-dev`, `<service-name>-prod` Application 생성.

---

## 4. 서비스별 빠른 참조 카드 (v2 — 결정 반영)

### 4.1 msa-product-service ✅ 완료

Phase 4 첫 PoC로 머지됨. 단순 추출 패턴 검증.

### 4.2 msa-order-service (Q5 + Q6 후속)

- **난이도**: 중
- **예상 작업**: 1~2시간
- **이미 폴리레포에 있음** (Phase 3). 후속 작업으로 변경:
  - **Q5 토픽명**: `inventory-reserve-request-topic` → `order.pending`, `inventory-reserved-result-topic` → `order.inventory-reserved`
    - `application.yaml`의 `spring.kafka.topic.*` 키 + 사용처 (`@Value`, Publisher/Listener) 수정
  - **Q6 state machine 확장**: `OrderLineItemStatus` enum에 PAYMENT_PENDING / PAID / SHIPPING / SHIPPED / CONFIRMED / CANCELLED 추가. 상태 전이 시 `order.confirmed`/`order.cancelled` 이벤트 발행 (Outbox 패턴 일관성 — 새 Outbox 테이블 또는 기존 `OrderInventoryRequestOutbox`를 일반화)
- **공통-libs**: `common:0.3.0` 만. `events:0.3.0`은 코드 양립용으로 보유 (wire는 JSON)
- **막힐 가능성**: state machine 전이 검증 — `OrderLineItemStatus.checkTransitive()`에 새 상태 추가 필요

### 4.3 msa-inventory-service (Q5 + Q7)

- **난이도**: **상** (Event Sourcing + Spring Batch 신규)
- **예상 작업**: 반나절+
- **포트**: 8003 (HTTP) / 9003 (gRPC) + worker pod (포트 없음)
- **DB**: `inventory_db`
- **Redis**: `inventory-service-redis` (password 필요)
- **Kafka**: consumer `order.pending` + producer `order.inventory-reserved` (Q5)
- **모노레포 모듈**: `inventory/` + `inventory-service/` + `inventory-event/`
- **의존**:
  - `com.troica.msa:common:0.3.0`
  - **JitPack** `com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2` (D2)
  - settings.gradle.kts에 `maven { url = uri("https://jitpack.io") }` 추가
- **신규 작업 (Q7)**:
  - `inventory-event`를 append-only 이벤트 스토어로 (현재는 Outbox 형태 — 처리 완료 마킹 제거하고 모든 이벤트 영구 보관)
  - **Spring Batch worker** 분기 (`SPRING_PROFILES_ACTIVE=worker`) + 별도 K8s Deployment
  - `@EnableBatchProcessing` + Job/Step 정의 (이벤트 스토어 → projection 또는 read model 갱신)
  - `application-worker.yaml`: web off, gRPC port=-1, Spring Batch Job Launcher만 동작

### 4.4 msa-user-service (단순 추출 + 신규 패키징)

- **난이도**: 중 (Q2 결정으로 단순화 — identification 폐기)
- **예상 작업**: ~1시간
- **포트**: 8004 (HTTP) / 9004 (gRPC) 
- **DB**: `user_db` (신규)
- **모노레포 모듈**: `user/` **단독** (identification은 모노레포에서 제거됨, auth는 별도 auth-service로)
- **의존**: `com.troica.msa:common:0.3.0`
- **핵심 작업**: `@SpringBootApplication` 클래스 + `application.yaml` **신규 작성**. UserRestController @RestController adapter 작성. user 도메인의 password encoder 빈 활용 (`SecurityConfig`에 추가됨)
- **gRPC**: auth-service가 `:user` 라이브러리를 임포트하지만, 향후 service-to-service 통신을 위해 user-service도 gRPC 서버 노출 권장

### 4.5 msa-auth-service (신규 레포 D1 — 신규 추출)

- **난이도**: 중 (모노레포 코드 거의 그대로)
- **예상 작업**: ~1시간
- **포트**: 8005 (HTTP) / 9005 (gRPC)
- **DB**: `auth_db` (신규)
- **Redis**: `auth-service-redis` (신규, refresh token 캐시)
- **모노레포 모듈**: `auth/` + `auth-service/`
- **의존**:
  - `com.troica.msa:common:0.3.0`
  - **JitPack** client-redis (refresh-token 캐시용)
  - jjwt-api 0.12.6 + jjwt-impl + jjwt-jackson
  - `auth-service`는 내부적으로 `:user` 도메인도 의존 (SignUp 시 user 생성)
- **핵심 작업**: 모노레포 패턴 그대로 폴리레포 이전. JWT secret은 ExternalSecrets로 주입 (Phase 0 후 정식 등록)
- **gRPC**: `AuthService.{SignUp, SignIn, CheckValidity}` 노출 — api-gateway의 JwtHeaderCheckFilter가 호출

### 4.6 msa-api-gateway (Q1 (c) 혼합)

- **난이도**: 중~상 (혼합 패턴 + JWT 필터 + Spring Cloud 신규 도입)
- **예상 작업**: 2~3시간
- **포트**: 8100 (HTTP), 외부 진입점
- **모노레포 모듈**: `user-api-gateway/`
- **의존**:
  - `com.troica.msa:common:0.3.0`
  - **Spring Cloud BOM 2025.0.2** (Northfields)
  - `spring-cloud-gateway-server-webflux` (명시 아티팩트)
  - `grpc-client-spring-boot-starter` (auth-service / product-service / order-service / inventory-service / user-service 호출)
  - jjwt 라이브러리 (header parse 용도, validation은 auth-service에 위임)
- **혼합 패턴 (Q1 (c))**:
  - 기존 BFF REST controllers (UserInventoryApiGatewayRestController 등) 그대로 보존
  - **신규 추가**: `spring.cloud.gateway.routes`에 product-service / order-service / inventory-service / user-service / auth-service로의 reverse-proxy 라우팅 (BFF 미경유 path)
  - `JwtHeaderCheckFilter`로 `/api/v1/orders/**` 등 보호 경로 검증 — auth-service gRPC `CheckValidity` 호출 (모노레포 코드 그대로)

### 4.x ~~msa-notification-service~~ (폐기)

- D6 결정: GitHub Settings에서 Archive
- 시스템 알림은 Prometheus → AlertManager → Slack 경로 (Phase 5 platform)

---

## 5. 진행 순서 권장 (v2)

병렬 가능 그룹:

**그룹 A — 독립 작업 (병렬 시작 가능)**
- ① 본 문서 PR (이 PR)
- ② common-libs v0.3.0 (client-redis/client-ses 제거)
- ③ order-service Q5 토픽명 + Q6 state machine 확장

**그룹 B — 사용자 GitHub 액션 후**
- ④ inventory-service Phase 4 + Q5 + Q7 (Event Sourcing + Spring Batch)
- ⑤ user-service Phase 4
- ⑥ msa-auth-service Phase 4 (신규 레포 D1 생성 후)

**그룹 C — 의존성 있음 (순차)**
- ⑦ api-gateway Phase 4 (⑥ auth-service 머지 후 — gRPC client 대상)

각 서비스 PR set = 서비스 레포 PR 1개 + 매니페스트 레포 values PR 1개. 총 12 PR 예상.

---

## 6. 회피 체크리스트 (Phase 2~3 교훈)

| # | 사고 | Phase | 회피 방법 |
|---|------|-------|----------|
| 1 | `gradlew` mode 100644 → Linux CI permission denied | Phase 2 R-feedback | 신규 레포 staging 시 `git update-index --chmod=+x gradlew` |
| 2 | `aquasecurity/trivy-action@0.24.0` 미존재 + 공급망 사고 범위 | Phase 3 R-14 | `@v0.36.0` 핀, v prefix 필수 |
| 3 | common-libs `0.x.0-SNAPSHOT` 의존 → GH Packages에서 못 받음 | Phase 3 R-11/R-17 | PR 머지 직전 stable `0.x.0`으로 bump |
| 4 | Phase 0 미완 상태 main CI 영구 빨간 | Phase 3 R-19 | push-gated step에 `&& vars.AWS_DEPLOYMENTS_ENABLED == 'true'` 가드 |
| 5 | Kotlin 2.3.x + Gradle 9.x `BuildUtilKt` 버그 | Phase 3 R-16 | Kotlin 2.1.0 + Gradle 8.10.2 고정 |
| 6 | SB 3.3.0 EOL + CVE | Phase 3 R-18 | Spring Boot BOM 3.5.13 명시 |
| 7 | bootJar JarLauncher 경로 검증 안 하고 Dockerfile 작성 | Phase 3 R-06 | 빌드 후 `unzip -p ... META-INF/MANIFEST.MF` 로 `Main-Class` 확인 |

---

**문서 버전**: 1.0
**작성일**: 2026-05-12
**갱신**: Phase 4 진행 중 발견사항을 BACKLOG에 R 항목으로 추가하고 본 문서도 동기화
