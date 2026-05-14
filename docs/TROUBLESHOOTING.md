# Troica Troubleshooting

> Troica polyrepo 마이그레이션 (Phase 0~) 진행 중 마주친 이슈와 해결책 누적.
>
> - **BACKLOG.md**: 작업 진행 상태 (완료/진행 중/대기)
> - **본 문서**: 디버깅 자료 — 증상 / 원인 / 해결 / 재발 방지
>
> 같은 이슈 재발 시 본 문서 검색 → 빠른 복구.

## 목차

1. [빌드 / 의존성](#1-빌드--의존성)
2. [보안 / CVE](#2-보안--cve)
3. [컨테이너 빌드](#3-컨테이너-빌드)
4. [CI/CD](#4-cicd)
5. [Terraform / AWS 인프라](#5-terraform--aws-인프라)
6. [Windows / PowerShell](#6-windows--powershell)
7. [Kubernetes / ArgoCD](#7-kubernetes--argocd)
8. [kubelet / Ansible](#8-kubelet--ansible)

---

## 1. 빌드 / 의존성

### 1.1 gradlew permission denied (CI)

**증상**: GitHub Actions에서 `./gradlew build` 실행 시 `Permission denied (exit 126)`.

**원인**: Windows에서 만든 `gradlew`가 git index에 `100644`로 저장됨. Linux CI runner에서 실행 권한 없음.

**해결**: 첫 commit 전 한 번 실행:
```bash
git update-index --chmod=+x gradlew
git commit -m "fix(gradlew): mark executable"
```
이후 git에서 `100755`로 추적되어 어느 OS에서 clone해도 정상.

**재발 방지**: 신규 Gradle 레포 생성 시 첫 commit에 매번 적용. `MEMORY.md`에 등재됨.

---

### 1.2 Kotlin 2.3.x + Gradle 9.x `BuildUtilKt.clearJarCaches` NoClassDefFoundError

**증상**: `./gradlew build` 실행 시:
```
java.lang.NoClassDefFoundError:
  org/jetbrains/kotlin/incremental/ClasspathEntrySnapshotter$Settings
  at BuildUtilKt.clearJarCaches(BuildUtilKt.kt:...)
```

**원인**: Kotlin 2.3.20/2.3.21의 Build Tools API JAR 내부 클래스 참조 깨짐. JetBrains upstream 버그.

**시도한 워크어라운드 (모두 실패)**:
- `kotlin.compiler.runViaBuildToolsApi=false`
- `kotlin.incremental.useClasspathSnapshot=false`
- `kotlin.incremental=false`

**해결**: **Kotlin 2.1.0 + Gradle 8.10.2 고정.** 모노레포 toolchain(2.3.20/9.0.0) 사용 금지.

**재발 방지**: 모든 신규 서비스 레포의 `build.gradle.kts` + `gradle/wrapper/gradle-wrapper.properties`에서 버전 고정.

---

### 1.3 JitPack `client-redis` Kotlin metadata 호환

**증상**: inventory-service 빌드 시:
```
The class file ... was compiled by a newer version of Kotlin compiler than 2.1
```

**원인**: 팀장님이 JitPack에 게시한 `com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`가 Kotlin 2.3.x로 컴파일됨. 우리 컴파일러는 2.1.0.

**해결**: consumer 측 (inventory-service `build.gradle.kts`)에 컴파일러 인자 추가:
```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict", "-Xskip-metadata-version-check")
    }
}
```

**재발 방지**: JitPack 의존성 사용하는 모든 서비스에 동일 컴파일러 인자.

---

### 1.4 common-libs `0.x.0-SNAPSHOT` GitHub Packages 부재

**증상**: 서비스 CI에서 `Could not find com.troica.msa:common:0.x.0-SNAPSHOT`.

**원인**: common-libs는 git tag (`v0.x.0`) push 시 GH Packages에 publish됨. SNAPSHOT은 publish 안 됨.

**해결**: 머지 직전에 의존성 버전을 `0.x.0-SNAPSHOT` → `0.x.0` (stable)로 bump. common-libs는 tag push로 publish.

**재발 방지**: 패턴화됨. v0.1.0, v0.2.0, v0.3.0 모두 동일 절차로 처리.

---

### 1.5 Spring Batch ItemWriter Kotlin lambda 타입 추론 실패

**증상**: `inventory-service`의 Spring Batch worker 빌드 시:
```kotlin
Type mismatch: inferred type is (Chunk<...>) -> Unit but ItemWriter<...> was expected
```

**원인**: Kotlin SAM(single abstract method) 변환이 Spring Batch의 `ItemWriter` 제네릭 wildcard와 충돌.

**해결**: 람다 대신 `object : ItemWriter<...>` 명시:
```kotlin
object : ItemWriter<InventoryEventDomainEntity> {
    override fun write(chunk: Chunk<out InventoryEventDomainEntity>) {
        // ...
    }
}
```

---

### 1.6 inventory-service가 `:inventory-event` 모듈 transitive 미인식

**증상**: `inventory-service` 빌드 시 `Unresolved reference: InventoryEventDomainEntity`.

**원인**: 의존성 그래프상 transitive로 들어올 줄 알았으나 명시적 선언 필요.

**해결**: `inventory-service/build.gradle.kts`에 직접 추가:
```kotlin
implementation(project(":inventory-event"))
```

---

### 1.7 Kotlin `@Configuration` class가 final → Spring CGLIB proxy 실패 (R-38)

**증상**: Spring Boot startup 시 ApplicationContext 초기화 단계에서:
```
org.springframework.beans.factory.parsing.BeanDefinitionParsingException:
  Configuration problem: @Configuration class 'QuerydslConfig' may not be final.
  Remove the final modifier to continue.
```

Phase 0 demo 시 CrashLoopBackOff의 진짜 원인이었음 (DB 부재가 아니라 startup 자체 실패).

**원인**:
- Kotlin은 모든 class가 기본 **`final`**.
- Spring `@Configuration`은 `@Bean` 메서드 호출 가로채기 위해 **CGLIB subclass proxy** 생성 → non-final 필요.
- 모노레포에서는 Spring Boot main 모듈에 `kotlin-spring` plugin이 적용되어 transitive로 처리됐었음. polyrepo 이관 시 `common-libs:common` 모듈에 plugin 누락.

**해결**: `common-libs/common/build.gradle.kts`에 `kotlin("plugin.spring")` 추가:
```kotlin
plugins {
    kotlin("kapt")
    kotlin("plugin.jpa")
    kotlin("plugin.spring")    // 추가
}
```

이 plugin이 컴파일 시 `@Configuration` / `@Component` / `@Service` / `@Repository` / `@Controller` 어노테이션이 붙은 클래스에 자동으로 `open` 부여 → publish된 .class가 non-final.

**검증**: 새 버전 publish + consumer 서비스 의존성 bump 후 `docker run`에서:
- 이전: `BeanDefinitionParsingException: 'QuerydslConfig' may not be final` ← 여기서 종료
- 이후: `Bootstrapping Spring Data JPA repositories` → `Tomcat initialized` → `HikariPool-1 - Starting` → `Connection refused` (DB 없음, 정상)

**재발 방지**:
- 모든 신규 모듈 build.gradle.kts에 `kotlin("plugin.spring")` + (JPA entity 있으면) `kotlin("plugin.jpa")` 표준 적용.
- common-libs `events` 모듈처럼 Spring 비사용 모듈은 plugin 불필요.

---

### 1.8 JUnit Platform launcher 명시 누락 — `OutputDirectoryProvider not available` (R-57)

**증상** (CI / 로컬 모두):
```
org.junit.platform.commons.JUnitException: TestEngine with ID 'junit-jupiter'
  failed to discover tests
Caused by: org.junit.platform.commons.JUnitException: OutputDirectoryProvider
  not available; probably due to unaligned versions of the junit-platform-engine
  and junit-platform-launcher jars on the classpath/module path.
```

**원인**: Gradle 8.10.2 + JUnit Platform 1.12+ 조합에서 `OutputDirectoryProvider`
API 가 `junit-platform-launcher` 측으로 옮겨감. `spring-boot-starter-test` 는
`junit-platform-engine` 만 transitive 로 가져와 launcher 가 누락됨 → test
discovery 단계에서 fail.

**해결**:
```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

7 polyrepo 모두 동일 적용 (R-57). 단순 build 검증 시점에는 잡히지 않고 실제
test 실행 단계에서만 발생하므로 R-57 첫 PR 푸시에서 일괄 fail 후 보강 commit
으로 해결.

---

### 1.9 grpc-kotlin generated stub 2-인자 시그니처 (mockk 매칭 누락, R-57)

**증상** (api-gateway InventoryQueryServiceTest):
- 정상 경로 테스트가 fallback 값으로 fail:
  - `fetchInventory 정상 경로`: `Expected "P-42" but was ""`
  - `fetchAll 정상 경로`: `Expected size: 2 but was: 0`
- 같은 테스트의 fallback 경로 는 통과 — 정상 경로만 fallback 으로 빠짐

**원인**: grpc-kotlin 가 생성하는 coroutine stub 메서드 시그니처는 2 인자.
```kotlin
public suspend fun fetchInventory(
    request: FetchInventoryRequest,
    headers: Metadata = Metadata(),
): InventoryResponseDto
```

Kotlin 컴파일러는 호출 시 두 번째 default 인자를 항상 자동 전달 → 실 호출은
`stub.fetchInventory(req, Metadata())`. mockk 매처가 1 인자 만 등록한 경우
unmatched call → mockk 가 예외 throw → `runCatching` 이 catch → fallback 경로.

**해결**:
```kotlin
// 잘못된 1 인자 매처
coEvery { stub.fetchInventory(any<FetchInventoryRequest>()) } returns response

// 정정 — 2 인자 매처
coEvery { stub.fetchInventory(any<FetchInventoryRequest>(), any()) } returns response
```

**재발 방지**: generated code (Protobuf / gRPC / annotation processor 등) 시그니처
는 추측 금지. `build/generated/sources/proto/main/grpckt/.../*GrpcKt.kt` 의 실제
파일을 직접 확인.

---

## 2. 보안 / CVE

### 2.1 Trivy action `@0.24.0` 미존재 + 공급망 공격

**증상**: CI에서 `aquasecurity/trivy-action@0.24.0` → `Prepare all required actions` 단계 실패 (conditional 평가 전).

**원인**: 2026-03-19 trivy-action 공급망 사고로 태그 `0.0.1~0.34.2`가 모두 compromised. 사고 후 새 컨벤션은 `v` prefix.

**해결**: `@v0.36.0` (2026-04-22, post-incident)로 pin. SPEC §7 + 모든 6개 서비스 ci.yml 적용.

**재발 방지**: 향후 SHA pin + Dependabot 도입 권장 (R-15).

---

### 2.2 Spring Boot 3.3.0 CVE-2025-22235 + EOL

**증상**: 기술 스택 audit 중 Spring Boot 3.3.x EOL (2025-06-30) + CVE-2025-22235 (HIGH, CVSS 7.3) 발견.

**원인**: 모노레포가 SB 3.3.0 사용 중. polyrepo로 옮기면서 그대로 따라옴.

**해결**: SB 3.5.13으로 일괄 bump (common-libs + 모든 서비스). Spring Cloud 2025.0.2 (Northfields, SB 3.5.x 공식 페어) 호환 확인.

**잘못된 가정 정정**: 팀장님은 "3.5.14 호환되는 Spring Cloud 없다"고 함. 실제로는 2025.0.2가 SB 3.5.x 전체 호환. 2025.1.0이 SB 4.x 전용으로 헷갈렸음.

---

### 2.3 Trivy가 main push 첫 시도에 12개 HIGH CVE 차단

**상황**: Phase 0 `AWS_DEPLOYMENTS_ENABLED=true` 활성화 후 첫 ECR push 시도. Trivy 보안 게이트가 작동.

**잡힌 CVE**:
| CVE | Library | Fix | 처리 |
|---|---|---|---|
| CVE-2026-40973 | spring-boot 3.5.13 | 3.5.14 | dependency bump |
| CVE-2026-34483/34487 | tomcat-embed-core 10.1.53 | 10.1.54 | SB 3.5.14 BOM 자동 |
| CVE-2026-42198 | postgresql 42.7.10 | 42.7.11 | 명시적 override (SB BOM 미반영) |
| CVE-2025-55163 | grpc-netty-shaded 1.68.1 | 1.75.0 | grpc 명시 bump |
| CVE-2026-42583 | netty-codec 4.1.132 (Lettuce/Kafka에서) | 4.1.133 | 명시 override |
| CVE-2026-42579 | netty-codec-dns 4.1.132 | 4.1.133 | 명시 override |
| CVE-2026-42584/42587 | netty-codec-http(2) 4.1.132 | 4.1.133 | 명시 override (api-gateway) |
| CVE-2026-42577 | netty-transport-native-epoll 4.1.x | 4.2.13 only | `.trivyignore` accepted risk |
| CVE-2026-5598 | bcprov-jdk18on 1.80 | 1.84 | 명시 override |

**해결 결과**: 모든 CVE 해결 또는 정당화. CI 통과 → ECR push 성공.

**시연 가치**: "자동 보안 게이트가 prod로 가는 길목에서 12개 HIGH CVE 차단" — Tech UP 발표 핵심.

---

### 2.4 `.trivyignore` 명시적 accepted risk (api-gateway)

**상황**: CVE-2026-42577 (netty-transport-native-epoll DoS via RST) — fix가 netty 4.2.13.Final에만 있음 (4.1.x line은 fix 없음).

**원인**: reactor-netty 1.2.x (Spring Boot 3.5.x BOM)는 netty 4.1.x 기준. 4.2.x로 bump 시 ABI 호환 보장 안 됨.

**해결**: `.trivyignore`로 명시적 accepted risk. 위협 모델 근거 (private k8s + Istio Gateway 뒤 → 직접 TCP RST exploit 비현실적). 해제 조건은 R-28 (SB가 netty 4.2.x BOM 갱신 시).

---

### 2.5 평문 JWT secret 노출 (auth-service)

**증상**: `application.yaml`에 `secret: ${JWT_SECRET:v7S6A9yB2E5H8KcNfUjXnZr4u7x!A%D*G-}` — 평문 fallback이 GitHub public repo에 노출.

**원인**: 모노레포 PoC의 평문이 polyrepo로 그대로 따라옴.

**해결 (1차)**: fallback을 명백한 placeholder로 치환:
```yaml
secret: ${JWT_SECRET:DEV_ONLY_INSECURE_REPLACE_ME_WITH_REAL_SECRET}
```
**해결 (정식, Phase 5)**: ExternalSecrets Operator + AWS Secrets Manager.

---

## 3. 컨테이너 빌드

### 3.1 Alpine(musl) + protoc-gen-grpc-java glibc 비호환

**증상**: Docker build의 `:product-service:generateProto` 단계에서:
```
protoc-gen-grpc-java-1.68.1-linux-x86_64.exe:
program not found or is not executable
```

**오해의 소지**: "program not found"는 파일은 있지만 dynamic linker가 못 로드하는 상황.

**원인**: Dockerfile build stage가 `eclipse-temurin:21-jdk-alpine` (musl libc). Maven Central의 `protoc-gen-grpc-java` 바이너리는 glibc 동적 링크.

**해결**: build stage만 glibc 베이스로:
```dockerfile
# Before
FROM eclipse-temurin:21-jdk-alpine AS build

# After
FROM eclipse-temurin:21-jdk AS build      # Ubuntu/Debian, glibc
```
Runtime stage는 alpine 유지 → 최종 image 크기 그대로. 6개 서비스 모두 적용.

**왜 이제 발견됐나**: `AWS_DEPLOYMENTS_ENABLED=false`일 때 Docker build step 자체가 skip됨 → 게이트 활성화 첫 시도에 노출.

---

## 4. CI/CD

### 4.1 CI 영구 red — OIDC AssumeRole 실패

**증상**: Phase 3 머지 후 main 빌드가 OIDC role 없어서 영구 빨강. 팀이 "원래 빨갛다"에 익숙해질 위험.

**원인**: Phase 0 (OIDC role 생성) 전에 push-gated step이 무조건 실행됨.

**해결**: 모든 push-gated step에 게이트 추가:
```yaml
if: github.event_name == 'push' && vars.AWS_DEPLOYMENTS_ENABLED == 'true'
```
Phase 0 완료 + Org Variable `AWS_DEPLOYMENTS_ENABLED=true` 설정 시 자동 활성.

---

### 4.2 ECR `GetAuthorizationToken` rate limit (병렬 머지)

**증상**: 5개 서비스 동시 머지 후 3개에서 ECR login timeout:
```
net/http: request canceled while waiting for connection
(Client.Timeout exceeded while awaiting headers)
```

**원인**: GitHub Actions runner의 egress IP에서 5개 동시 `aws ecr get-login-password` → AWS가 source IP rate limit. Fresh account라 limit이 더 낮음.

**해결**: 머지를 직렬로 (한 번에 하나, 1-2분 간격). 또는 Re-run을 1개씩.

**재발 방지**: R-27 (f) CI 워크플로우에 ECR login retry wrapper 추가 권장.

---

### 4.3 CI 빌드 ~4분 → ~2.5분 (R-27 (a) 적용 후)

**개선 전 구성 요소**:
- Gradle 호스트 빌드 (테스트 포함): ~1.5분
- Docker build (Gradle 내부 재실행): ~1.5분 (중복!)
- Trivy DB 다운로드 (첫 회): ~1분 (vuln-db 92MB + java-db 865MB)
- ECR push + manifest bump: ~30s

**R-27 (a) 적용 (2026-05-12)**: Docker의 build stage 제거 → 호스트 bootJar 결과를 layered extract만. **product-service 검증: 4분 → 2분 25초**.

**개선 후**:
- Gradle 호스트 빌드: ~1.5분 (그대로)
- Docker build (layered extract만): ~10s (Gradle 재실행 없음)
- Trivy DB: ~30s (캐시 hit 시 ~5s)
- ECR push + manifest bump: ~30s

**잔여 개선 여지** (R-27, Phase 5+):
- (b) Docker BuildKit cache mount → 호스트 빌드의 Gradle cache hit 시 -30~60s
- (c) Trivy cache key 주 단위 → -1분 (daily first-run only)
- (d) Reusable workflow — 6 ci.yml DRY (시간 절감보다는 유지보수)

---

### 4.4 Dependabot 활성화 직후 자동 PR 폭주 (R-27 (g) 폐기 사유)

**증상**: 활성화 첫 주에 7+ 자동 PR 일괄 생성:
- aws-actions/configure-aws-credentials 4 → 6
- peter-evans/create-pull-request 7 → 8
- actions/setup-java 4 → 5
- docker/build-push-action 6 → 7
- gradle-wrapper 8.10.2 → 9.5.0  ← 우리 toolchain 결정(R-09)과 충돌
- actions/checkout 4 → 6
- spring-boot-dependencies (build fail)

**평가**:
- review 부담 큰 반면 빌드 시간 절감과 무관 (R-27 본질에서 벗어남).
- Spring Boot BOM이 transitive 의존성 다수 관리 → 대부분 BOM 버전 bump만 잡음.
- 실제 CVE는 이미 Trivy가 빌드 단계에서 게이트.
- Gradle 9.x bump는 Kotlin 2.1 toolchain 호환성 깨짐 → 잘못된 방향 권장.

**결정**: Dependabot 폐기. 추후 Renovate (정교한 정책 가능) 또는 monthly check 검토.

**참고**: GitHub의 "Automatically delete head branches" 설정도 같이 켜두면 자동 PR close 시 stale 브랜치 자동 정리.

---

### 4.5 SonarCloud 무료 plan — custom Quality Gate 적용 불가 + PR scan main ref 부재 (R-45)

**증상 1** (무료 plan 제약):
SonarCloud Project 페이지의 Quality Gate 화면:
> Your current plan does not allow you to associate a Quality Gate other than
> Sonar way (Default) to this project. Upgrade plan.

Organization 차원에서 custom Gate 생성은 가능하나 `Set as Default` 버튼이 paid
feature 로 잠김. default Sonar way 의 "New Code Coverage ≥ 80%" 조건이 강제 적용
됨.

**증상 2** (PR scan main ref 부재):
```
Could not find ref: main in refs/heads, refs/remotes/upstream or refs/remotes/origin
Shallow clone detected, no blame information will be provided.
File 'X.kt' not found in project sources
```

GitHub Actions `checkout@v4` default `fetch-depth: 1` → main ref 없음 → SonarCloud
가 PR diff 비교 불가 → file source 인식도 일부 깨짐.

**원인 (Gate fail)**: PR scan 의 new code = build.gradle.kts 변경 + test 파일.
test 파일은 coverage 측정 대상이 아니므로 new code coverage = 0% → Sonar way
gate 항상 fail. 닭-달걀 상황 (main 머지 후 main scan 에서야 coverage 측정됨).

**해결**: ADR-0011 — `sonar.qualitygate.wait = false` 일괄 적용.
```kotlin
sonar {
    properties {
        property("sonar.qualitygate.wait", "false")
    }
}
```

- 분석은 정상 수행 → 결과는 Dashboard 에 push
- CI 는 Gate 평가 결과를 기다리지 않고 통과
- Hotspot Review 는 SonarCloud UI 에서 처리 (Safe / Acknowledged / Fixed)

**후속**: 단위 테스트 정착 (R-57 후속 + coverage 점진 향상) 또는 paid plan 전환
시 `wait=true` 복구. fetch-depth: 0 옵션 추가는 별도 PR 검토.

---

### 4.6 Trivy manifest scan workflow 누적 fix 3건 (R-46)

`msa-argocd-manifest` 의 `.github/workflows/trivy-manifest-scan.yml` 가 main 머지 후
3 번 연속 fail. 각 회 다른 원인 — 사후 정리.

#### 4.6.1 `if:` 표현식 내 `secrets.*` 직접 참조 불가 (PR #80)

**증상**:
```
##[error]Unrecognized named-value: 'secrets'.
Located at position 17 within expression: failure() && secrets.SLACK_SECURITY_WEBHOOK_URL != ''
```

**원인**: GitHub Actions 보안 정책 — `if:` 표현식 안에서 `secrets.*` 직접 참조 금지.
secret 존재 여부를 step `env:` 또는 별도 boolean job output 으로 노출 후에야 가능.

**해결**: `if: failure() && secrets.X != ''` → `if: failure()` + `continue-on-error: true`.
webhook 미설정 상태에서는 slack-github-action 이 빈 webhook 으로 fail 하나
`continue-on-error` 가 흡수하여 workflow 전체에는 영향 X.

#### 4.6.2 action 태그 v prefix 누락 — `Unable to resolve action 0.36.0` (PR #82 → #83 흡수)

**증상** (PR #83 첫 CI 시도):
```
##[error]Unable to resolve action `aquasecurity/trivy-action@0.36.0`,
unable to find version `0.36.0`
```

**원인**: aquasecurity/trivy-action 의 release tag 는 v prefix (`v0.36.0`).
2026-03-19 trivy-action 공급망 사고 이후 v prefix 컨벤션 (R-14 / [TROUBLESHOOTING §2.1](#21-trivy-action-0240-미존재--공급망-공격)).
다른 polyrepo (msa-product-service 등) 의 ci.yml 은 `@v0.36.0` 으로 정확히 작성되어
있었으나 R-46 작업 시 일관성 누락.

**해결**: `@0.36.0` → `@v0.36.0`. PR #82 가 본 fix 만 담고 있었으나 §4.6.3 도 동시에
필요해서 PR #83 에 흡수 후 #82 close (superseded).

#### 4.6.3 `exit-code: "1"` 가 PoC 단계 매니페스트 46건과 충돌 (PR #83)

**증상** (action 정상 resolve 후):
```
2026-05-14T08:39:41Z  INFO  Detected config files num=46
Error: Process completed with exit code 1.
```

**원인**: K8s 표준 룰셋이 SecurityContext / runAsNonRoot / readOnlyRootFilesystem /
resources / probes 누락 등을 HIGH/CRITICAL 로 검출. PoC 단계 매니페스트는 의도적으로
이들을 미보강 상태 — `exit-code: "1"` 게이트가 매니페스트 보강 작업과 무관한 PR 까지
전부 차단.

**해결**: PoC 단계 정보 수집 모드로 전환.
```yaml
# Before
exit-code: "1"
# After
exit-code: "0"   # SARIF 는 그대로 GitHub Security 탭에 업로드됨
```

SARIF 출력 + `upload-sarif` step 은 그대로 유지 → 위반 목록은 GitHub Security 탭에서
확인 가능. step 자체는 성공 처리 → CI 통과.

**운영 전환 시점**: SecurityContext / resources 등 매니페스트 보강을 마친 후 별도 PR
로 `exit-code: "0"` → `"1"` 승격. Phase 6 (B) 운영 전환 항목.

**재발 방지**:
- 신규 보안 스캔 workflow 도입 시 PoC 단계는 정보 수집 모드, 운영 전환 시 게이트 모드
  의 2 단 운영 결정.
- action 태그 작성 시 다른 polyrepo 의 동일 action 사용처 grep 후 prefix 일치 확인.
- `if:` 표현식 안 secrets 참조 시도 자체를 피하고 `continue-on-error` 패턴 사용.

---

## 5. Terraform / AWS 인프라

### 5.1 cloud-provider-aws release에 binary asset 없음

**증상**: ansible `get_url`로 `ecr-credential-provider-linux-amd64` 다운로드 → 404.

**원인**: cloud-provider-aws v1.30.x release 페이지의 `assets`가 0개. AWS EKS AMI에 미리 설치되어 있어서 binary 별도 배포 안 함.

**시도한 버전 점검**:
- v1.30.5 (우리가 처음 시도) — release 자체 없음
- v1.30.10 (1.30 라인 최신) — release 있지만 assets=0

**해결**: Go 소스에서 직접 빌드 → ansible로 노드에 copy:
```bash
sudo apt install -y golang-go
git clone --depth 1 --branch v1.30.10 https://github.com/kubernetes/cloud-provider-aws.git
cd cloud-provider-aws
CGO_ENABLED=0 go build -o /tmp/ecr-credential-provider ./cmd/ecr-credential-provider
```
ansible playbook이 `/tmp/ecr-credential-provider`를 노드로 copy.

---

### 5.2 terraform fmt 오류 — JSON StringLike 줄 분리

**증상**: `terraform fmt -check` 실패 — JSON `StringLike` 내 줄 분리 인식 안 됨.

**해결**: 해당 JSON 값을 한 줄로 합침.

---

### 5.4 PVC 동적 생성 EBS 가 cluster destroy 후 orphan — 비용 누수 (R-60)

**증상**: terraform destroy-temp 완료 후 AWS 콘솔 EBS 콘솔에서 `available` 상태
EBS volume 다수 발견. tag `kubernetes.io/created-for/pvc/name` 가짐. 시간당 비용
계속 발생.

**원인**: PVC 로 동적 생성된 EBS 는 terraform 관리 X. EBS CSI driver 가 PVC 삭제
시 EBS 도 자동 정리하나, **cluster 가 destroy 되면 CSI controller 도 사라짐**
→ orphan PVC 와 그에 연결된 EBS 가 AWS 에 남음.

**해결**: `scripts/destroy-temp.ps1` 에 자동 cleanup task 추가 (PR #16):
```powershell
$orphans = aws ec2 describe-volumes `
    --filters Name=status,Values=available Name=tag-key,Values=kubernetes.io/created-for/pvc/name `
    --query 'Volumes[].VolumeId' --output text --region ap-northeast-2
$orphans -split '\s+' | Where-Object { $_ } | ForEach-Object {
    aws ec2 delete-volume --volume-id $_ --region ap-northeast-2
}
```

추가로 destroy 끝에 비용 sanity check — EBS / Snapshot / LoadBalancer / Unassociated
EIP / NAT Gateway / EFS 6 종류 개수 출력 (0 = green, 0 아니면 red).

**재발 방지**:
- 신규 stateful workload 도입 시 PVC 동적 EBS 가 새 종류 자원 만드는지 확인
- destroy-temp 의 sanity check 가 다음 사이클 자원 종류 확장 trigger
- 새 종류 추가 (RDS / EFS CSI / ALB 등) 시 destroy-temp cleanup 도 함께 PR

---

### 5.5 Worker 노드 사양 부족 — CoreDNS 가 master 로 schedule + service pod fail (R-59)

**증상**: 첫 cluster up (worker × 3 t3.medium 4 GB) 후 ArgoCD root-app SYNC=Unknown,
application-controller log:
```
Reconnect to redis because error: "dial tcp: lookup argocd-redis: i/o timeout"
ComparisonError: failed to generate manifest ... dns: lookup argocd-repo-server
on 10.96.0.10:53: dial udp i/o timeout
```

**진단 (먼 길)**: 처음에 Calico IPIP tunnel / SG / DNS UDP 등 의심하며 5 회 진단
사이클. 결정적 단서 — `kube-prometheus-stack-prometheus-node-exporter` 가
**`Evicted`** 상태. 메모리 압박 신호.

**근본 원인**: worker × 3 t3.medium = 12 GB 총. Phase 5 매니페스트 (Prometheus
1-2 GB + Kafka 3 GB + Postgres × 6 × 2 = 3 GB + Istio + Loki/Tempo + 6 polyrepo
× 2 env ...) 합산 18-20 GB 예상. 부족.

CoreDNS 가 worker 우선이어야 하나 worker 메모리 부족으로 master (taint 무시
toleration) 에 schedule → master ↔ worker pod 사이 service 트래픽 timeout (실
원인 미규명, 메모리 압박과 관련 추정).

**해결**: msa-provisioning PR #13. worker 만 t3.large (8 GB) 로 변경. master 는
t3.medium 유지 (control plane only). 비용 +$0.156/h.

**재발 방지**:
- 새 stateful workload 추가 시 worker capacity 합산 확인
- `kubectl top nodes` + `kubectl describe nodes | grep Allocatable` 로 사전 점검
- Pending pod 의 events 가 `Insufficient memory` 면 사양 부족 확정

---

### 5.3 PR 2가 PR 1 자원 참조 → terraform validate 실패

**증상**: PR 2의 `aws_iam_role_policy_attachment`가 PR 1의 `data.aws_iam_role` 참조. PR 2 단독 validate 실패.

**해결**: stacked PR — PR 2를 PR 1 branch에 rebase. PR 1 머지 후 PR 2가 main 기준으로 정렬.

---

## 6. Windows / PowerShell

### 6.1 PowerShell `-flag=value` 인수 파싱 깨짐

**증상**:
```
terraform plan -out=phase-0.tfplan
→ Error: Too many command line arguments
```

**원인**: PowerShell 5.1이 native exe에 `-flag=value`를 두 개 인수로 쪼갬.

**해결 옵션**:
- 공백 분리: `terraform plan -out phase-0.tfplan`
- 인용부호: `terraform plan "-out=phase-0.tfplan"`
- Stop-parsing 토큰: `terraform plan --% -out=phase-0.tfplan`
- Array splatting (여러 개일 때):
  ```powershell
  $t = @("-target=...", "-target=...")
  terraform destroy @t -auto-approve
  ```

---

### 6.2 PowerShell JSON quoting (aws s3api)

**증상**:
```
aws s3api put-bucket-encryption ... --server-side-encryption-configuration '{"Rules": [...]}'
→ Invalid JSON: Expecting property name enclosed in double quotes
```

**원인**: PowerShell 5.1이 single-quoted JSON을 native exe에 전달 시 따옴표 보존 깨짐.

**해결**: AWS CLI shorthand 사용 (JSON 회피):
```powershell
aws s3api put-bucket-encryption `
  --bucket $BUCKET `
  --server-side-encryption-configuration "Rules=[{ApplyServerSideEncryptionByDefault={SSEAlgorithm=AES256}}]"
```

---

### 6.3 `.sh` 파일이 git에서 CRLF로 저장 → bash 실패

**증상**:
```bash
bash destroy-temp.sh
→ $'\r': command not found
→ set: pipefail: invalid option name
```

**원인**: Windows의 git `core.autocrlf` 기본값이 `.sh`를 CRLF로 변환.

**해결**:
- 단기: PowerShell 네이티브 `destroy-temp.ps1` 추가 (별도 PR로 머지됨).
- 영구: `.gitattributes`에 `*.sh text eol=lf` 강제. Windows에서 clone해도 워킹 디렉토리는 LF.

---

### 6.4 `aws s3api put-public-access-block`의 잘못된 파라미터명

**증상**:
```
Unknown parameter in PublicAccessBlockConfiguration: "RestrictPublicAccess"
must be one of: BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets
```

**원인**: 정확한 이름은 `RestrictPublicBuckets`. AWS docs 작성 시 흔한 오타.

**해결**: 명령어의 `RestrictPublicAccess=true` → `RestrictPublicBuckets=true`.

---

### 6.5 destroy-temp.ps1 한글 깨짐 — PS 5.1 parser CP949 디코딩 (R-60)

**증상** (destroy 실행 시):
```
==> Plan (?꾩떆 ?먯썝留?destroy)
... (정상 동작은 함)
==> ?꾨즺. ?곴뎄 ?먯썝(OIDC/IAM/ECR/KMS)?
```

**1차 시도 (PR #14, 실패)**: 스크립트 시작에 console output encoding 강제.
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
$null = chcp 65001
```
→ 여전히 깨짐.

**근본 원인**: PowerShell 5.1 의 **parser stage** 가 스크립트 파일을 시스템 코드
페이지 (한국 환경 = CP949) 로 디코드 후 메모리 로드. 파일은 UTF-8 (no BOM) 인데
parser 가 ANSI 로 해석 시도 → 한글 byte 가 다른 character 로 매핑된 채로 메모리에
들어감. 그 후 `[Console]::OutputEncoding=UTF8` / `chcp 65001` 가 콘솔 출력 stage 의
encoding 만 바꾸므로 **이미 깨진 메모리 문자열은 그대로** 출력.

**해결 옵션**:
- UTF-8 BOM 추가 — PS 5.1 parser 가 BOM 인식 후 UTF-8 로 처리. 단 git workflow 에서
  BOM 누락 위험 (`.gitattributes` 로 강제 필요)
- ASCII (영어) 만 사용 — 가장 안전. 어떤 환경에서도 작동

**채택**: PR #16 — 모든 user-facing 한글 string 을 영어로 전환. ASCII safe.
`destroy-temp.sh` (bash) 는 parser 가 LANG 따라가므로 한글 그대로 유지.

**재발 방지**:
- PowerShell 5.1 호환 필요한 스크립트는 ASCII 만 사용
- 한글 메시지가 정말 필요하면 UTF-8 BOM + `.gitattributes` 의 `*.ps1 working-tree-encoding=UTF-8-BOM`

---

## 7. Kubernetes / ArgoCD

### 7.1 ArgoCD `directory.include` brace expansion 미동작

**증상**: `bootstrap/root-app.yaml`의:
```yaml
directory:
  include: '{projects/x.yaml,platform/y.yaml,applications/z.yaml}'
```
→ 4개 파일 중 0개 매칭. 자식 Application 0개 생성.

**원인**: ArgoCD가 `glob.Glob` 패턴만 지원. brace expansion `{a,b,c}`는 매칭 안 함.

**해결**: app-of-apps 정식 패턴으로 전환.
- `bootstrap/root-app.yaml`: path=bootstrap, recurse=false
- `bootstrap/apps.yaml` 신규: multi-doc으로 3개 자식 Application

**동일 패턴 fix**: `platform/root.yaml`의 `{*/application.yaml,*/*/application.yaml}` → `**/application.yaml` doublestar.

---

### 7.2 root-app이 팀장님 개인 레포 가리킴

**증상**: 클러스터 부트스트랩 후 `argocd app list`에서:
```
root-app    https://github.com/kanei0415/ktcloud-k8s-argocd-manifest.git    Setup
```
우리 팀 매니페스트가 무시됨.

**원인**: `ansible/argocd-setup.yaml`이 root Application을 inline yaml로 정의하면서 팀장님 PoC 레포 URL hardcoded.

**해결**: ansible playbook이 우리 매니페스트 레포의 `bootstrap/root-app.yaml`을 fetch + apply하는 패턴으로 전환. 매니페스트가 단일 진실의 원천.

---

### 7.3 ArgoCD auto-sync가 "all resources를 wipe out" 가드로 멈춤

**증상**: 새 root-app이 `Synced` 안 되고:
```
Skipping sync attempt: auto-sync will wipe out all resources
```

**원인**: 이전 root-app이 남긴 `charts-app` ApplicationSet이 orphan으로 살아있음. 새 root-app과 desired state diff가 "전부 wipe" → ArgoCD 안전 가드 발동.

**해결**: orphan ApplicationSet 수동 제거 후 재sync:
```bash
kubectl -n argocd delete applicationset charts-app addons-app apps-app 2>/dev/null
argocd app sync root-app
```

---

### 7.4 pod `CreateContainerConfigError` — Secret/ConfigMap 미존재

**증상**: pod 상태 `CreateContainerConfigError` + events:
```
Error: secret "api-gateway-secrets" not found
```

**원인**: Helm chart의 `envFrom`이 `{service}-secrets`, `{service}-config` 참조하지만 외부에서 생성된 적 없음.

**해결 (PoC)**: placeholder Secret/ConfigMap 일괄 생성:
```bash
for svc in api-gateway auth-service user-service product-service order-service inventory-service; do
  for ns in market-dev market-prod; do
    kubectl create secret generic ${svc}-secrets -n $ns --from-literal=PLACEHOLDER=tbd
    kubectl create configmap ${svc}-config -n $ns --from-literal=PLACEHOLDER=tbd
  done
done
```

**정식 해결 (Phase 5)**: ExternalSecrets Operator + AWS Secrets Manager (R-33).

**현상 정정**: 위 임시 fix로 pod는 startup 가능하지만 DB 없어서 Spring Boot가 `CrashLoopBackOff`. 이게 Phase 0의 자연스러운 경계.

---

### 7.5 K8S_VARS_HOLDER unreachable

**증상**: ansible playbook 실행 중:
```
fatal: [K8S_VARS_HOLDER]: UNREACHABLE!
```

**원인**: `join-master.yaml`에서 vars 저장용 가상 호스트. inventory에 없는 상태.

**영향**: 다른 노드의 task는 모두 정상. 가상 호스트의 `gather facts` 실패만 발생. **무시 가능**.

---

### 7.6 ArgoCD AppProject sourceRepos 와 Helm chart repoURL mismatch (PR #87)

**증상**: cluster up 후 다수 Application 이 `SYNC=Unknown`. conditions:
```
InvalidSpecError: application repo https://kubernetes-sigs.github.io/aws-ebs-csi-driver
is not permitted in project 'platform'
```

**원인**: ArgoCD 의 `AppProject.spec.sourceRepos` 가 보안 제약 (allow-list). 새 helm
chart 를 platform Application 에 추가하면서 `projects/platform-project.yaml` 의
sourceRepos 갱신을 누락. R-35 작성 시 4 개 sourceRepo 누락 발견:
- `https://kubernetes-sigs.github.io/aws-ebs-csi-driver` (EBS CSI)
- `https://charts.external-secrets.io` (ESO)
- `https://spotahome.github.io/redis-operator` (sourceRepos 는 `ot-container-kit` 잘못 등록)
- `https://strimzi.io/charts/` (trailing slash mismatch)

ArgoCD 의 sourceRepos 매칭은 **URL 정확 일치** (trailing slash 까지).

**해결**: `projects/platform-project.yaml` 의 sourceRepos 에 4 개 추가 / 교체.

**재발 방지**:
- 새 helm chart application 추가 시 `projects/<project>-project.yaml` 의 sourceRepos
  도 동시 PR 에 포함
- helm repo URL 의 trailing slash 매니페스트 ↔ AppProject 일치 확인

---

### 7.7 ArgoCD selfHeal=true 가 cluster 측 kubectl patch 를 즉시 되돌림 (Lesson)

**증상**: §7.6 디버깅 중 `kubectl -n argocd patch appproject platform --type json -p
'[{"op":"add","path":"/spec/sourceRepos/-","value":"..."}]'` 으로 sourceRepo 4 개
추가. 30 초 후 application 상태 확인 시 여전히 `InvalidSpecError` + sourceRepos
개수 7 (추가 전 8 보다 더 적음 — 다른 항목까지 reset).

**원인**: `platform-root` Application 의 `spec.syncPolicy.automated.selfHeal=true`.
ArgoCD 가 cluster state (우리 patch) ↔ Git main (옛 sourceRepos) 비교 시 자동으로
main desired state 로 cluster 강제 동기화 → 우리 patch 즉시 무효화.

**해결**: cluster 측 kubectl patch 대신 **Git main 머지 후 ArgoCD 자동 reconcile**
대기. selfHeal 작동하는 환경에서 cluster 즉시 fix 는 의미 없음.

**디버깅 진행 시 lesson**:
- ArgoCD application status conditions message 가 명확한 원인 (`InvalidSpecError`,
  `ComparisonError`) 을 직접 알려주면 그것부터 따라감
- cluster 측 patch 시도 전 `selfHeal` 여부 확인 (`kubectl get application <name> -o
  jsonpath='{.spec.syncPolicy.automated.selfHeal}'`)

---

### 7.8 self-managed kubeadm 에서 EBS CSI driver 미설치 → StorageClass 0 개 (PR #86 + PR #15)

**증상**: cluster up 후 모든 PVC `Pending`:
```
$ kubectl get pvc -A
... 10 PVCs all Pending (CNPG × 6 + Grafana/Prometheus/Loki/Tempo)

$ kubectl get sc
No resources found

$ kubectl describe pod prometheus-... | grep Events
Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims
```

**원인**: EKS 와 달리 self-managed kubeadm cluster 는 EBS CSI driver 가 자동 설치
되지 않음. PVC 가 binding 할 StorageClass 가 0 개.

**해결** (2 PR):
1. **msa-argocd-manifest PR #86** — `platform/05-ebs-csi/application.yaml` 추가.
   helm chart `aws-ebs-csi-driver v2.35.1`, Sync Wave -9 (cert-manager 다음).
   gp3 StorageClass (default, encrypted, `WaitForFirstConsumer`).
2. **msa-provisioning PR #15** — EC2 node IAM role 에 AWS managed
   `AmazonEBSCSIDriverPolicy` 부착. EBS CSI controller 가 IMDS → EC2 node IAM 으로
   AWS API (CreateVolume / AttachVolume / etc) 호출.

**재발 방지**:
- self-managed cluster 의 platform setup checklist 에 EBS CSI driver 포함
- EKS 가정 매니페스트 도입 시 IRSA → instance profile 인증으로 전환 필요

---

### 7.9 Helm chart values.schema.json strict — Istio gateway 1.23 `defaults:` wrapping (PR #88)

**증상** (`istio-cp` Application):
```
ComparisonError: helm template ... failed:
gateway:
- at '': additional properties 'replicaCount', 'resources', 'service', 'defaults' not allowed
```

**원인**: Istio gateway helm chart **1.23 의 `values.schema.json`** 는 top-level 에
`defaults:` wrapper 만 허용 (`additionalProperties: false`). chart values.yaml 주석
은 "`--set replicaCount=X` 로 직접 설정 가능" 이라고 안내하나 — 그건 helm CLI 의
`--set` flag 자동 flatten 일 때만. ArgoCD 처럼 **values yaml 통째로 전달 시 schema
그대로 적용** → reject.

**해결**: 매니페스트 values 를 `defaults:` 안으로 nesting.
```yaml
# Before
service:
  type: NodePort
replicaCount: 1

# After
defaults:
  service:
    type: NodePort
  replicaCount: 1
  autoscaling:
    enabled: false   # chart default true 라 PoC 에 불필요
```

**검증**: `helm pull istio/gateway --version 1.23.3 --untar` 후 `values.schema.json`
의 root keys 직접 확인 — `'$ref'` + `'defaults'` 만 있고 root properties 없음.

**재발 방지**:
- 새 helm chart 도입 시 `helm pull --untar` + `values.schema.json` 의 root structure
  먼저 확인
- ArgoCD `spec.source.helm.skipSchemaValidation: true` 는 임시 우회용 (PoC), 운영
  단계에는 정식 schema 따라야 함

---

### 7.10 ExternalSecret target.name 과 service Helm envFrom 이름 mismatch → CreateContainerConfigError (R-25/R-33, PR #89)

**증상**: cluster up 후 모든 service pod `CreateContainerConfigError`:
```
$ kubectl describe pod api-gateway-... | grep -A 3 Events
... MountVolume.SetUp failed for volume "...": secret "api-gateway-secrets" not found
```

ExternalSecret 은 정상 작동 (`SecretSynced=True`).

**원인**: ExternalSecret 이 만든 secret 의 이름 (`auth-db-credentials` in databases ns,
`auth-jwt-secret` in market-dev) 과 service Helm chart 가 envFrom 으로 참조하는
이름 (`{svc}-secrets`, `{svc}-config` in market-{dev,prod}) 이 다름. R-35 매니페스트
작성 시 ExternalSecret target.name 과 service Helm chart 의 envFrom 이 따로 따로
작성됨.

**해결**: PR #89 — service 별 ExternalSecret + ConfigMap 12 쌍 추가.
- ExternalSecret `target.name={svc}-secrets` (service Helm 기대 이름)
- `target.template.data` 로 ENV 이름 매핑 (`AUTH_DB_PASSWORD: "{{ .dbPassword }}"`)
- ConfigMap `{svc}-config` — K8s service DNS (CNPG `{name}-rw`, Strimzi
  `market-kafka-kafka-bootstrap`, Spotahome `rfr-{name}`)

각 polyrepo 의 `application.yaml` 의 `${ENV:default}` 패턴 직접 추출 후 정확히 매핑.

**재발 방지**:
- service Helm chart 의 `envFrom` 참조 이름 = ExternalSecret `target.name` 일치 검증
- ArgoCD sync 후 `kubectl describe pod` 의 events 가 `secret ... not found` 면 즉시 이
  패턴 의심

---

### 7.11 ClusterSecretStore IRSA → IMDS 인증 변경 (self-managed kubeadm, PR #85)

**증상** (cluster up 전 매니페스트 분석 단계 발견):
```yaml
spec:
  provider:
    aws:
      auth:
        jwt:
          serviceAccountRef:        # EKS IRSA 패턴
            name: external-secrets
            namespace: external-secrets
```

매니페스트 주석은 "kubelet instance profile (R-29 와 동일 IRSA 패턴)" 이라고 정확히
기술했으나 spec 은 IRSA — 모순.

**원인**: IRSA (IAM Roles for Service Accounts) 는 EKS 의 OIDC provider 가 필요.
self-managed kubeadm cluster 에는 OIDC provider 부재 → STS `AssumeRoleWithWebIdentity`
호출 실패 → 모든 ExternalSecret sync 실패.

**해결**: PR #85 — `spec.provider.aws.auth` 섹션 완전 제거. ESO operator pod 가
AWS SDK default credential provider chain 으로 **IMDS → EC2 node IAM role** 자동
사용 (R-29 ECR credential provider 와 동일 패턴).

사전 조건: msa-provisioning iam.tf 에 `secretsmanager:GetSecretValue` 부착 (PR #12,
R-25/R-33 의 짝).

**재발 방지**:
- 매니페스트 주석 vs spec 일관성 검증 — 코드 직접 읽기 (사용자 작업 원칙)
- self-managed cluster 의 모든 AWS API 호출 컴포넌트 (ESO / EBS CSI / ALB controller
  / etc) 는 IRSA 가 아닌 IMDS 패턴 사용
- 첫 cluster up 전 ClusterSecretStore / 다른 IAM 사용 매니페스트 의 auth section 점검

---

## 8. kubelet / Ansible

### 8.1 kubelet `KUBELET_EXTRA_ARGS` drop-in 미평가

**증상**: ansible playbook으로 drop-in 작성:
```ini
# /etc/systemd/system/kubelet.service.d/30-ecr-credential-provider.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--image-credential-provider-config=..."
```
`systemctl show kubelet --property=ExecStart`에 `$KUBELET_EXTRA_ARGS` 보이지만, `ps aux | grep kubelet`에는 ECR 인자 미반영.

**원인**: Amazon Linux 2023 + kubeadm 1.30 환경에서 systemd가 `Environment="KUBELET_EXTRA_ARGS=..."`를 어떤 이유로 평가 안 함. 근본 원인 미규명.

**해결**: `/var/lib/kubelet/kubeadm-flags.env`의 `KUBELET_KUBEADM_ARGS`에 ECR 인자를 직접 추가:
```ini
KUBELET_KUBEADM_ARGS="--image-credential-provider-config=... --image-credential-provider-bin-dir=/usr/local/bin --container-runtime-endpoint=... --pod-infra-container-image=..."
```
playbook의 lineinfile task로 통합. drop-in은 다른 환경 호환 위해 유지.

**검증**: `ps aux | grep kubelet`에 ECR 인자 보이면 성공. ansible playbook에 자동 검증 task 추가:
```yaml
- name: kubelet args에 ECR provider 인자 실제 반영 확인
  shell: ps -ef | grep '[k]ubelet' | grep -q image-credential-provider-config
  failed_when: result.rc != 0
```

---

### 8.2 Docker image pull 시 `no basic auth credentials`

**증상**: pod events:
```
Failed to pull image "601766312629.dkr.ecr.ap-northeast-2.amazonaws.com/msa/api-gateway:..."
pull access denied, repository does not exist or may require authorization:
authorization failed: no basic auth credentials
```

**원인**: kubelet에 ECR credential provider 인자가 안 들어감 (위 8.1 참조). credential provider가 호출되지 않아서 ECR 인증 토큰 미발급.

**해결**: 8.1 fix 적용 후 kubelet 재시작 → ECR pull 정상 작동.

**검증 신호**: events에 `Successfully pulled image ... Image size: XXXMb` 보이면 ECR credential provider 정상.

---

## 부록 — 유용한 디버깅 명령

### kubelet
```bash
# kubelet 실제 args 확인
ps aux | grep '[k]ubelet'

# kubelet drop-in 적용 상태
sudo systemctl show kubelet --property=Environment --property=ExecStart

# kubelet 로그 (특정 키워드)
sudo journalctl -u kubelet --since "10 minutes ago" | grep -iE "credential|ecr|pull"
```

### ArgoCD
```bash
# 모든 application 상태
argocd app list
kubectl -n argocd get application,applicationset,appproject

# 특정 app 디버깅
argocd app get <app-name> --refresh
kubectl -n argocd describe application <app-name>
```

### Pod
```bash
# pod 이벤트 + 상세
kubectl describe pod -n <ns> <pod>
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -30

# ConfigMap/Secret 참조 확인
kubectl get pod -n <ns> <pod> -o yaml | grep -A 2 -E "configMap|secret|envFrom"
```

### Trivy 로컬 검증
```bash
trivy image <image-ref> --severity HIGH,CRITICAL --ignore-unfixed
```
