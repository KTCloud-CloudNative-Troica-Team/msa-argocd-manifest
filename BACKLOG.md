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
| R-11 | GitHub Packages publish 실 검증 | Phase 2 | 로컬 publishToMavenLocal은 성공. 원격 GitHub Packages publish는 `v0.1.0` 태그 푸시 시점에 검증 필요. PR 머지 후 사용자 수행 | Phase 2 종료 직전 |
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
