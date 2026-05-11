# Troica MSA 작업 백로그

> **참조**: 모든 작업은 [TROICA_SPEC.md](./TROICA_SPEC.md)의 Phase 구조를 따른다.
> 결정 변경 시 SPEC도 함께 갱신하고 본 문서의 **SPEC 변경 이력** 섹션에 한 줄 기록한다.
>
> **작업 원칙**
> - `main` 직접 커밋 금지. 모든 변경은 feature branch + PR.
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

---

## SPEC 변경 이력

(없음 — 초안 v1.0 그대로 시작)

---

## 보류 / 검토 필요 (SPEC 1차 리뷰 결과)

SPEC v1.0 정독 + 기존 레포 상태 대조 결과 발견한 사항. 각 Phase 진입 시점에 결정 반영.

| # | 항목 | 위치 | 사안 | 결정 시점 |
|---|------|------|------|----------|
| R-01 | `values-worker.yaml` 별도 파일 미사용 | SPEC §2 디렉토리 트리 | 차트 §3.4는 `.Values.worker.enabled` 플래그로 분기. 별도 `values-worker.yaml` 파일은 어디서도 로드되지 않음 → 사실상 dead file | Phase 1: 트리에서 제거하거나 ApplicationSet에서 명시 로드 |
| R-02 | App-of-Apps multi-source 스니펫 오류 | SPEC §6 | `source:` 와 `sources:` 동시 정의는 ArgoCD validation에서 reject. 둘 중 하나만 사용 | Phase 5: ArgoCD v2.6+ 확인 후 `sources:` 단독으로 수정 |
| R-03 | Traefik vs Istio 충돌 | `msa-provisioning` 현 상태 vs SPEC §3.2 | 현 클러스터는 Traefik 사용 중. SPEC 차트는 `istio.enabled: true` default + `VirtualService` 템플릿. Istio 미설치 상태에서 차트 배포하면 sidecar inject annotation만 무해하게 무시되지만, 일관성 위해 결정 필요 | Phase 1: 차트 default를 `istio.enabled: false`로 두고 Phase 5에서 Istio platform Application 추가 시 toggle |
| R-04 | `user-api-gateway` 명칭 불일치 | 모노레포 module명 vs SPEC §1.1 | SPEC: `msa-api-gateway` / 기존 모노레포: `user-api-gateway` | Phase 4 진입 시 신규 레포명은 `msa-api-gateway`로, 내부 Gradle module은 리네이밍 또는 유지 결정 |
| R-05 | `notification-service`는 그린필드 | SPEC §1.1, §9 vs 모노레포 | 모노레포에 notification 도메인 코드 없음. 완전 신규 구현 | Phase 4: 별도 트랙으로 일정 산정 |
| R-06 | Spring Boot 버전 검증 필요 | SPEC §8 Dockerfile 주석 | `JarLauncher` 경로가 SB 3.2부터 `org.springframework.boot.loader.launch.JarLauncher`로 변경. 현 모노레포 SB 버전 미확인 | Phase 3: 첫 Dockerfile 작성 직전 `unzip -l build/libs/*.jar` 로 manifest 확인 |
| R-07 | `order-worker` profile 분기 존재 여부 | SPEC §1.1, §3 vs 코드 | order-service 코드에 `worker` profile 기반 Outbox poller 분기가 실제로 구현돼있는지 미확인 | Phase 3: 코드 이전 시 함께 검증, 없으면 별도 작업 추가 |
| R-08 | `msa-spring-boot` 아카이브는 마이그레이션 후 | SPEC §1.3 주석 | 아카이브는 모든 서비스 이전 완료 후. 이전 중에는 read-only 금지 | Phase 6 |

---

## 누적 결정 사항 (SPEC §0 외 추가)

(없음)
