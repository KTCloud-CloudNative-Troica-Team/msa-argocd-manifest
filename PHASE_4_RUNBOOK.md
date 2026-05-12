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

> P0 답변 미수령이면 진행 가능한 서비스: **product-service**(질문 무관), 그 다음 **inventory-service**(Q4/Q5/Q7 영향)는 답 받은 후.

---

## 1. 모노레포 → 폴리레포 매핑

| 폴리레포 | 가져갈 모노레포 모듈 | deployable app? | 비고 |
|----------|---------------------|---------------|------|
| `msa-product-service`   | `product` + `product-service` | ✅ `ProductServiceApplication` 존재 | 단순 추출. 가장 빠름 |
| `msa-inventory-service` | `inventory` + `inventory-service` + `inventory-event` | ✅ `InventoryServiceApplication` 존재 | Redis 사용 + Outbox(`inventory-event`) + Kafka. Q4/Q5/Q7 영향 |
| `msa-user-service`      | `user` + `auth` + `identification` | ❌ **모노레포에 없음 — 신규 패키징** | Q2 답에 따라 분리 가능성. `@SpringBootApplication` 작성 필요 |
| `msa-api-gateway`       | `user-api-gateway` | ✅ `UserApiGatewayApplication` 존재 | Q1 답에 따라 BFF 유지 / SC Gateway 리라이트 |
| `msa-notification-service` | (그린필드) | ❌ 코드 0줄 | Phase 0 후. SPEC §9 참조해서 신규 구현 |

---

## 2. 포트 / DB / 외부 의존성 매트릭스

### 2.1 네트워크 포트

| 서비스 | HTTP/REST | gRPC | 비고 |
|--------|-----------|------|------|
| product-service       | 8001 | 9001 | |
| order-service         | 8002 | 9002 | worker pod는 `web-application-type=none` + `grpc.server.port=-1` (포트 없음) |
| inventory-service     | 8003 | 9003 | |
| user-service          | **8004** | **9004 또는 미사용** | 신규. Q1/Q2에 따라 gRPC 필요 여부 결정 |
| notification-service  | **8005** | - | Kafka consumer 전용. HTTP는 actuator만 |
| api-gateway           | 8100 | - | 외부 진입점 |

> 8000번대 차감 / 9000번대는 gRPC. 모노레포 컨벤션 그대로 + user/notification 신규 할당.

### 2.2 PostgreSQL / Redis / Kafka / 외부 API

| 서비스 | PostgreSQL DB | Redis | Kafka | SES | Slack |
|--------|--------------|-------|-------|-----|-------|
| product-service       | `product_db` | - | - | - | - |
| order-service         | `order_db` | - | producer + consumer | - | - |
| inventory-service     | `inventory_db` | `inventory-service-redis` (require pass) | producer + consumer | - | - |
| user-service          | `user_db` (신규) | `user-service-redis` (신규, auth refresh-token 캐시) | - | - | - |
| notification-service  | `notification_db` (신규, 멱등 테이블) | - | consumer만 | ✅ | ✅ |
| api-gateway           | - | - | - | - | - |

> 모노레포 `container-compose.yaml`은 product/order/inventory + Kafka + inventory-redis만 정의. user/notification은 신규 추가 필요.

### 2.3 GitHub Packages 의존성 (common-libs 모듈 단위)

| 서비스 | `:common` | `:client-redis` | `:client-ses` | `:events` |
|--------|:---------:|:---------------:|:-------------:|:---------:|
| product-service       | ✅ | - | - | - |
| order-service         | ✅ | (간접) | - | ✅ |
| inventory-service     | ✅ | ✅ | - | ✅ |
| user-service          | ✅ | ✅ (auth refresh-token 캐시) | - | - |
| notification-service  | ✅ | ✅ (멱등 처리) | ✅ | ✅ |
| api-gateway           | (간접) | - | - | - |

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

## 4. 서비스별 빠른 참조 카드

### 4.1 msa-product-service

- **난이도**: 낮음 (단순 추출)
- **예상 작업**: 30분
- **포트**: 8001 (HTTP) / 9001 (gRPC)
- **DB**: `product_db`
- **Kafka/Redis**: 미사용
- **모노레포 모듈**: `product/` + `product-service/`
- **의존**: `com.troica.msa:common:0.2.0`
- **막힐 가능성**: 거의 없음. order-service 패턴 복붙

### 4.2 msa-inventory-service

- **난이도**: 중 (Outbox + Redis 분산락 + Kafka)
- **예상 작업**: 1~2시간 (Q4 Protobuf 마이그레이션 포함 시)
- **포트**: 8003 (HTTP) / 9003 (gRPC)
- **DB**: `inventory_db`
- **Redis**: `inventory-service-redis` (password 필요)
- **Kafka**: producer + consumer (Q5 토픽명 결정에 따라)
- **모노레포 모듈**: `inventory/` + `inventory-service/` + `inventory-event/`
- **의존**: `com.troica.msa:common:0.2.0` + `client-redis:0.2.0` + `events:0.2.0`
- **막힐 가능성**: Protobuf 마이그레이션 — order-service 측 코드도 동시 변경 필요. **양쪽 PR set으로 진행**
- **Q4 답이 (b)면 추가 작업**: 양쪽 KafkaConfig.kt + Mapper 2개씩 + Message 클래스 삭제 + application.yaml의 serializer 설정

### 4.3 msa-user-service

- **난이도**: 중~상 (deployable app 신규 작성)
- **예상 작업**: 1시간 (Q2 (a) 통합) ~ 2시간 (Q2 (b) 분리)
- **포트**: 8004 (HTTP) / 9004 (gRPC 필요한지 결정)
- **DB**: `user_db` (신규)
- **Redis**: `user-service-redis` (신규, auth refresh-token 캐시용)
- **모노레포 모듈**: `user/` + `auth/` + `identification/`
- **의존**: `com.troica.msa:common:0.2.0` + `client-redis:0.2.0`
- **핵심 작업**: `@SpringBootApplication` 클래스 + `application.yaml` **신규 작성**, AuthRestController/UserRestController/ExternalIdentificationRestController 인터페이스에 `@RestController` 어노테이션 + 구현 어댑터 클래스 추가 (모노레포에는 인터페이스만)
- **Q3 JWT 검증 강제 활성화**: api-gateway 측 작업. 본 서비스는 그저 발급/리프레시 엔드포인트만 제공

### 4.4 msa-api-gateway

- **난이도**: Q1 결정에 따라
  - **(a) BFF 유지**: 낮음 (~30분 단순 추출)
  - **(b) SC Gateway 리라이트**: 중~상 (~수 시간, routes 설계)
  - **(c) 둘 다**: 별도 서비스로 분리
- **포트**: 8100 (HTTP)
- **DB / Redis / Kafka**: 미사용
- **모노레포 모듈**: `user-api-gateway/`
- **의존**: Spring Cloud 2025.0.2 BOM (api-gateway 한정)
- **Q1 (b) 선택 시 추가 작업**:
  - `spring-cloud-gateway-server-webflux` 아티팩트 명시 (legacy `spring-cloud-starter-gateway` 가능하나 비추)
  - `application.yaml`에 `spring.cloud.gateway.routes` 정의 (product/order/inventory/user-service로 라우팅)
  - JWT 인증 필터 작성 (Q3 답에 따라 강제 vs permitAll)
  - Rate Limit 필터 (Redis token bucket)
  - BFF의 REST/gRPC 변환 코드는 폐기 또는 별도 서비스로 분리

### 4.5 msa-notification-service

- **난이도**: 상 (그린필드)
- **예상 작업**: 반나절+
- **Phase 0 의존**: SES/Slack credentials 필요 → **Phase 0 후로 보류 결정됨**
- **포트**: 8005 (HTTP, actuator만)
- **DB**: `notification_db` (신규, 멱등성 테이블)
- **Kafka consumer**: `order.confirmed`, `order.cancelled`, `notification.requested` (Q5 답에 따라 토픽명)
- **모노레포 모듈**: 없음. **그린필드**
- **의존**: `com.troica.msa:common:0.2.0` + `client-redis:0.2.0` + `client-ses:0.2.0` + `events:0.2.0`
- **사전 결정 필요** (Q8): channel 분기, 멱등성 키 저장소, DLQ 토픽명

---

## 5. 진행 순서 권장

P0 답변 미수령 상태에서 시작 가능:
1. **product-service** — 결정 무관, 패턴 검증용
2. (P0 답 받은 후) **inventory-service** — Protobuf 마이그레이션과 묶음 (order-service 변경도 동시)
3. **user-service** — Q2 답 받은 후 통합/분리 결정
4. **api-gateway** — Q1 답 받은 후 BFF/SC Gateway 결정
5. **notification-service** — Phase 0 완료 후

각 서비스 PR set은 (a) 서비스 레포 PR 1개 + (b) 매니페스트 레포 values PR 1개 = 2 PR. 5개 서비스 × 2 = 10 PR 예상.

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
