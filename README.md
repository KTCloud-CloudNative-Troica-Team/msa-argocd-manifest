# msa-argocd-manifest

Troica Market Service MSA의 GitOps 매니페스트 레포.

> 모든 결정·구조의 단일 진실 소스는 [TROICA_SPEC.md](./TROICA_SPEC.md).
> 진행 상황은 [BACKLOG.md](./BACKLOG.md) 참조.

## 디렉토리 구조

```
.
├── TROICA_SPEC.md          # 청사진 (단일 진실 소스)
├── BACKLOG.md              # 작업 백로그 + SPEC 변경 이력
├── bootstrap/
│   ├── root-app.yaml       # kubectl apply 진입점
│   └── README.md
├── projects/               # ArgoCD AppProject CRD
│   ├── market-project.yaml
│   └── platform-project.yaml
├── applications/           # 마이크로서비스 레이어 (ApplicationSet)
│   ├── appset.yaml
│   ├── charts/microservice/  # 공통 Helm 차트
│   └── values/               # 서비스별 + 환경별 values
└── platform/               # 플랫폼 레이어 (App-of-Apps)
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
- 결정 변경 시 SPEC.md + BACKLOG.md 동시 갱신.
- 새 서비스 추가는 `applications/values/<service-name>/` 폴더 생성만으로 자동 등록.
- 새 플랫폼 컴포넌트는 `platform/<NN>-<name>/application.yaml` 추가만으로 자동 등록.
