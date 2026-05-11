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

---

## SPEC 변경 이력

| 일자 | 위치 | 변경 | 사유 |
|------|------|------|------|
| 2026-05-12 | §2 디렉토리 트리 | `order-service/values-worker.yaml` 라인 제거 + worker 분기는 flag로 한다는 주석 추가 | R-01 해결: dead file 제거 |
| 2026-05-12 | §6 multi-source 예시 | `source:` + `sources:` 중복을 `sources:` 단독으로 정리 | R-02 해결: ArgoCD validation 통과 |

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
| P1-V | Phase 1 클러스터-사이드 검증 미수행 | 로컬 환경 | 로컬에 ArgoCD CRD 설치된 클러스터 미연결 → kubectl dry-run 불가. 오프라인 YAML 구조 검증만 통과 | Phase 0/2 인프라 준비 후 실 클러스터에서 `kubectl apply -f bootstrap/root-app.yaml --dry-run=server` 수행 |

---

## 누적 결정 사항 (SPEC §0 외 추가)

| 일자 | 결정 | 근거 |
|------|------|------|
| 2026-05-12 | Istio가 최종 ingress/service mesh. Traefik은 Phase 5에서 제거 | 사용자 명시 |
| 2026-05-12 | `msa-spring-boot` 레포는 read-only로만 다룬다 (다른 작업자 충돌 회피) | 사용자 명시 |
| 2026-05-12 | Phase 진행 순서를 SPEC §13의 0 → 1 에서 **1 → 0 → 2 → ...** 로 조정 | Phase 0(Terraform apply)이 비가역이라 골격을 먼저 완성하고 컨펌 후 진행. 의존 깨지지 않음 |
| 2026-05-12 | ApplicationSet에 `_*` exclude 패턴 추가 — `_template` 디렉토리 도입 | 신규 서비스 추가 시 복사 시작점 + Phase 1 helm template 검증 대상. SPEC에는 미반영 (구현 디테일) |
