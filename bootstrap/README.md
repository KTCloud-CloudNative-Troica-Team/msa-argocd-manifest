# Bootstrap

ArgoCD에 본 매니페스트 레포를 등록하는 진입점.

## 적용 순서

### 0. 사전 조건

- 클러스터에 ArgoCD가 설치돼있고 `argocd` namespace가 존재한다.
- 본 레포가 public이거나, private이면 `kubectl -n argocd create secret ...` 으로 repo credential을 등록해뒀다.

### 1. 단일 명령으로 전체 sync

```bash
kubectl apply -f bootstrap/root-app.yaml
```

이 한 줄이 다음을 순서대로 sync한다 (sync-wave 사용):

| Wave | 리소스 | 비고 |
|------|--------|------|
| -10 | `AppProject/market`, `AppProject/platform` | `projects/*.yaml` |
| 0   | `Application/platform-root` | `platform/root.yaml` — Phase 1엔 sync 대상 없음 |
| 0   | `ApplicationSet/market-services` | `applications/appset.yaml` — Phase 1엔 generator 결과 0건 |

### 2. 상태 확인

```bash
argocd app list
argocd app get root-app
kubectl get appproject -n argocd
kubectl get applicationset -n argocd
```

## 추가/삭제 시

- **새 마이크로서비스** → `applications/values/<service-name>/{values.yaml,values-dev.yaml,values-prod.yaml}` 추가. ApplicationSet이 자동으로 `<service>-dev`, `<service>-prod` Application 생성.
- **새 플랫폼 컴포넌트** → `platform/<NN>-<name>/application.yaml` 추가. `platform-root`가 자동 sync.
- **레포 권한 확장** → `projects/*.yaml`의 `sourceRepos` / `destinations` 수정.
