# Platform layer

App-of-Apps + sync-wave 기반의 플랫폼 컴포넌트 매니페스트.

## 구조 (Phase 5에서 채워짐)

```
platform/
├── root.yaml                               # App-of-Apps root (이미 생성됨)
├── 00-cert-manager/application.yaml        # syncWave: -10
├── 10-istio-base/application.yaml          # syncWave: -5
├── 11-istio-cp/application.yaml            # syncWave: -4
├── 30-kube-prometheus-stack/
│   ├── application.yaml
│   └── values.yaml
├── 30-loki/
├── 30-tempo/
├── 30-mimir/
├── 40-strimzi-operator/
├── 40-cnpg-operator/
├── 40-redis-operator/
├── 50-kafka-cluster/
│   ├── kafka.yaml
│   └── topics/
├── 50-postgres-clusters/
└── 50-redis-cluster/
```

## Sync wave 규칙

음수일수록 먼저 실행. CRD → operator → 컴포넌트 → 의존 컴포넌트 순.
폴더명 prefix는 시각적 가이드이고 실제 순서는 Application CRD의 `argocd.argoproj.io/sync-wave` annotation으로 강제한다.

## Phase 1 시점

`root.yaml` 외에는 비어있다. `platform-root` Application은 sync 대상 0건으로 Healthy 상태가 정상.
