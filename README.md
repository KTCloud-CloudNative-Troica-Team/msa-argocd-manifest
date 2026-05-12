# msa-argocd-manifest

Troica Market Service MSA의 GitOps 매니페스트 레포.

> 모든 결정·구조의 단일 진실 소스는 [docs/TROICA_SPEC.md](./docs/TROICA_SPEC.md).
> 진행 상황은 [docs/BACKLOG.md](./docs/BACKLOG.md). 디버깅 자료는 [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md).
> 주요 아키텍처 결정 기록은 [docs/adr/](./docs/adr/).

## 디렉토리 구조

```
.
├── docs/                       # 문서 (SPEC / BACKLOG / TROUBLESHOOTING / ADR)
│   ├── TROICA_SPEC.md          # 청사진 (단일 진실 소스)
│   ├── BACKLOG.md              # Phase별 task 진행 상태
│   ├── TROUBLESHOOTING.md      # 누적 디버깅 자료
│   ├── PHASE_4_RUNBOOK.md      # Phase 4 작업 절차
│   ├── PROGRESS_LOG.md         # 컨텍스트 스냅샷
│   └── adr/                    # Architecture Decision Records
├── bootstrap/
│   ├── root-app.yaml           # kubectl apply 진입점 (app-of-apps)
│   ├── apps.yaml               # 3개 자식 Application multi-doc
│   └── README.md
├── projects/                   # ArgoCD AppProject CRD
│   ├── market-project.yaml
│   └── platform-project.yaml
├── applications/               # 마이크로서비스 레이어 (ApplicationSet)
│   ├── appset.yaml
│   ├── charts/microservice/    # 공통 Helm 차트
│   └── values/                 # 서비스별 + 환경별 values
└── platform/                   # 플랫폼 레이어 (App-of-Apps)
    └── root.yaml
```

## 부트스트랩

```bash
kubectl apply -f bootstrap/root-app.yaml
```

자세한 절차는 [bootstrap/README.md](./bootstrap/README.md).

## 로컬 검증

```bash
# Helm 차트 렌더링
helm template applications/charts/microservice \
  -f applications/values/_template/values.yaml \
  -f applications/values/_template/values-dev.yaml

# 매니페스트 lint
kubectl apply --dry-run=client -f bootstrap/root-app.yaml
kubectl apply --dry-run=client -f projects/
kubectl apply --dry-run=client -f applications/appset.yaml
kubectl apply --dry-run=client -f platform/root.yaml
```

## 변경 규칙

- `main` 직접 커밋 금지. 모든 변경은 feature branch + PR.
- **아키텍처 결정 변경 시**: SPEC.md + BACKLOG.md 동시 갱신 + 새 ADR 추가 (`docs/adr/NNNN-*.md`).
- 새 서비스 추가는 `applications/values/<service-name>/` 폴더 생성만으로 자동 등록.
- 새 플랫폼 컴포넌트는 `platform/<NN>-<name>/application.yaml` 추가만으로 자동 등록.
