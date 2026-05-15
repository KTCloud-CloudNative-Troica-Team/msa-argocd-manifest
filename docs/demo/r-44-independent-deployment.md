# R-44 (B) Demo — 독립 배포 + E2E 무영향 검증

> **평가 cover**: 심화 (1)-1 (필수) — 독립 배포 + 다른 서비스 트래픽 무영향
>
> **ADR**: [ADR-0009 — api-gateway ↔ Istio Service Mesh 책임 분담](../adr/0009-api-gateway-istio-mesh-collaboration.md)
>
> **소요**: ~15 분

## 사전 조건

cluster up + R-42 (B) 통과 (Newman + Grafana 작동 상태).

```bash
# service pod Running
kubectl -n market-dev get pods

# Grafana port-forward 준비 (별도 터미널)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &
```

## 평가 demo 의 본질

> **"한 service 만 변경 → 다른 service 트래픽 영향 0"**

product-service 만 rolling restart 도중/후, **auth / user / order / inventory / api-gateway** 의 다른 endpoint 응답이 정상 동등해야 함.

---

## Part 1 — Baseline (정상 상태) 측정

### Step 1 — Newman E2E baseline (다른 service endpoint)

```bash
# product-service 제외한 endpoint 응답 측정. 단순화 위해 healthz 호출.
kubectl run baseline -n market-dev \
    --image=postman/newman:6.2.1 \
    --restart=Never \
    --rm -i \
    --command -- /bin/sh -c '
        for svc in auth-service user-service order-service inventory-service; do
            for port in 8005 8004 8002 8003; do : ; done
            case $svc in
                auth-service) PORT=8005 ;;
                user-service) PORT=8004 ;;
                order-service) PORT=8002 ;;
                inventory-service) PORT=8003 ;;
            esac
            echo "=== $svc ==="
            for i in 1 2 3; do
                time curl -s -o /dev/null -w "  status=%{http_code} time=%{time_total}s\n" \
                    http://$svc.market-dev.svc.cluster.local:$PORT/healthz
            done
        done
    '
```

**기대**: 각 service 의 healthz status=200 + time ~수 ms (baseline 응답 시간 기록).

### Step 2 — Grafana 대시보드 baseline 캡쳐

브라우저 http://localhost:3000 → **Troica — 서비스 SLI 대시보드**

스크린샷 4 panel:
- 요청 처리량 (req/s) — 6 service 곡선
- 에러율 (5xx) — 모두 0%
- P99 응답 시간
- JVM Heap

발표 시 baseline 비교용.

---

## Part 2 — product-service 만 rolling deploy

### Step 1 — ArgoCD selfHeal 일시 비활성 (CRITICAL)

ArgoCD selfHeal=true 가 product-service 의 spec 변경 즉시 원복 → demo fail.

```bash
kubectl -n argocd patch application product-service-dev --type merge \
    -p '{"spec":{"syncPolicy":{"automated":null}}}'
```

> **주의**: Part 3 Step 4 에서 복원 필수.

### Step 2 — Rolling restart 발행

```bash
# product-service deployment 의 pod 재시작 (image 변경 없이 rolling update trigger).
# 매니페스트의 strategy: maxSurge=1, maxUnavailable=0 라 zero-downtime.
kubectl -n market-dev rollout restart deploy/product-service
```

**진정한 "image bump" demo 가 필요하면** (선택):
```bash
# 다른 ECR tag 로 변경 (ECR 에 해당 tag 가 있어야 함)
kubectl -n market-dev set image deploy/product-service \
    app=601766312629.dkr.ecr.ap-northeast-2.amazonaws.com/msa/product-service:main-<other-sha>
```

### Step 3 — Rollout 진행 watch

별도 터미널에서:
```bash
# product-service pod 변화 watch (1 → 2 → 1)
kubectl -n market-dev get pods -l app.kubernetes.io/name=product-service -w
```

**기대**:
1. 새 pod (`product-service-XXXX`) `0/2 Pending` → `0/2 ContainerCreating` → `1/2 Running` (sidecar 만 ready) → `2/2 Running` (app ready)
2. 옛 pod (`product-service-YYYY`) `2/2 Running` → `2/2 Terminating` → 사라짐
3. 항상 ≥ 1 pod available (`2/2 Running`) → **zero-downtime**

`Ctrl+C` 로 watch 종료. `rollout status` 로 완료 확인:
```bash
kubectl -n market-dev rollout status deploy/product-service
# 기대: deployment "product-service" successfully rolled out
```

---

## Part 3 — 다른 service 영향 측정

### Step 1 — Rollout 도중 다른 service 호출 (실시간)

**Part 2 Step 2 를 다시 실행하기 직전부터, 백그라운드 측정 시작**:

```bash
# 별도 터미널 — 30 초 동안 1 초마다 다른 service healthz 호출 (응답 시간 측정)
kubectl run probe -n market-dev \
    --image=postman/newman:6.2.1 \
    --restart=Never \
    --rm -i \
    --command -- /bin/sh -c '
        for i in $(seq 1 30); do
            echo "=== sec $i ==="
            for svc_port in "auth-service:8005" "user-service:8004" "order-service:8002" "inventory-service:8003"; do
                svc=$(echo $svc_port | cut -d: -f1)
                port=$(echo $svc_port | cut -d: -f2)
                curl -s -o /dev/null -w "$svc=%{http_code}(%{time_total}s) " \
                    http://$svc.market-dev.svc.cluster.local:$port/healthz
            done
            echo ""
            sleep 1
        done
    '
```

이 명령어 실행 직후, **다른 터미널에서 Part 2 Step 2 의 rollout 발행**.

**기대**: 30 초 동안 4 service 모두 `status=200` + `time ≈ baseline` (응답 시간 변화 < 10%).

product-service 만 영향 받고, 다른 4 service 는 무영향 ✅.

### Step 2 — Grafana 대시보드 비교

브라우저 http://localhost:3000 (rollout 완료 후 5 분 대기):

**Panel 1 (요청 처리량)**:
- product-service: rollout 동안 일시 낮아짐 → 회복
- 다른 service: **그래프 변화 없음** ✅

**Panel 2 (에러율)**:
- 모두 0% 유지

**Panel 3 (P99 응답)**:
- product-service: 새 pod startup 시 일시 spike → 회복
- 다른 service: 변화 없음

### Step 3 — Grafana 스크린샷 비교

Part 1 baseline 캡쳐 vs 현재 — 다른 service 그래프 동등 확인.

### Step 4 — selfHeal 복원 (CRITICAL)

```bash
kubectl -n argocd patch application product-service-dev --type merge \
    -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'

# 확인
kubectl -n argocd get application product-service-dev -o jsonpath='{.spec.syncPolicy}{"\n"}'
```

---

## 평가 발표 포인트

| 항목 | 시연 |
|---|---|
| **독립 배포** | product-service 만 rolling restart — 다른 service Deployment 영향 X |
| **E2E 무영향** | Part 3 Step 1 — 30 초 측정 동안 4 service status=200 유지 |
| **Zero-downtime** | strategy: maxSurge=1/maxUnavailable=0 → product 도 항상 ≥ 1 pod available |
| **ADR-0009** | Istio mesh = 인프라 layer / Resilience4j = 응용 layer 책임 분리. rolling 시 sidecar 가 새 pod 의 healthz 통과 후에 traffic forward |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| selfHeal patch 후 즉시 restart 도 rollback | application controller 처리 시간 — 5 초 대기 후 재시도 |
| 새 pod 0/2 (sidecar 못 떠) | namespace istio-injection=enabled 확인 + istiod Healthy |
| 다른 service 응답 시간 증가 | (a) Istio sidecar 의 DestinationRule outlierDetection 이 떨어지면서 routing 변경 (b) NodePort 부하 — 별 영향 없으면 OK |
| Newman pod 가 fail | image pull (postman/newman) 실패 — NAT 통한 docker.io 도달 확인 |
| Part 3 Step 1 의 `kubectl run probe` 가 이미 사용 중 | `kubectl -n market-dev delete pod probe --ignore-not-found` 후 재시도 |

---

## 검증 한 번에 통과 보장 — 정적/논리/실행/실패 흐름 검증 결과

| 단계 | 결과 |
|---|---|
| 정적 | helm template — strategy.rollingUpdate.maxSurge=1, maxUnavailable=0 정확 rendering |
| 논리 | replica=1 시 maxSurge=1 → 새 pod 1 ready 후 옛 pod 종료 = 항상 1+ pod. zero-downtime |
| 실행 | rollout restart → 새 ReplicaSet → 새 pod 2/2 → 옛 pod terminating → 완료 |
| 실패 | selfHeal 비활성 not 원복 위험 → Step 4 강조. probe pod 이름 충돌 → delete 후 재시도. sidecar 미주입 → namespace label 확인 |

---

## Reference

- [Deployment strategy maxSurge/maxUnavailable docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [ADR-0009 책임 분담](../adr/0009-api-gateway-istio-mesh-collaboration.md)
- [R-42 (B) Newman 측정 방법](r-42-e2e-and-monitoring.md)
