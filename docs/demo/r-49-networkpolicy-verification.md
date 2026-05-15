# R-49 (B) Demo — NetworkPolicy 통신 차단 검증

> **평가 cover**: 심화 (3)-3 (필수) — NetworkPolicy + 차단 테스트
>
> **매니페스트**: `platform/60-network-policies/manifests/market-dev.yaml` (Sync Wave 5)
>
> **소요**: ~10 분

## 사전 조건

cluster up + Sync Wave 5 (network-policies) 완료.

```bash
# market-dev 의 6 NetworkPolicy 등록 확인
kubectl -n market-dev get networkpolicy

# 기대 출력:
#   default-deny-ingress
#   api-gateway-allow-ingressgateway
#   backend-grpc-allow-gateway
#   user-service-allow-gateway
#   order-service-admin-allow-gateway
#   prometheus-scrape-allow

# Calico CNI 확인 (NetworkPolicy enforcer)
kubectl -n kube-system get pods -l k8s-app=calico-node
# 기대: 6 노드 모두 Running
```

---

## 평가 demo 의 본질

> **"의도된 통신만 허용, 의도하지 않은 통신은 차단"**

| 통신 경로 | NetworkPolicy 결과 | 의도 |
|---|---|---|
| istio-ingressgateway → api-gateway:8100 | ✅ allow | 외부 사용자 트래픽 |
| api-gateway → auth-service:9005 (gRPC) | ✅ allow | JWT 검증 호출 |
| **product-service → auth-service:9005** | ❌ **deny** | **inter-backend 호출 금지 (BFF 아키텍처)** |
| monitoring → service:8005 (metrics scrape) | ✅ allow | Prometheus scrape |
| 임의 pod (label 없음) → service | ❌ deny | default-deny-ingress |

---

## Part 1 — 허용 경로 검증 (Allow)

### Step 1 — api-gateway → auth-service:9005 (gRPC) 허용

```bash
# api-gateway 의 sidecar 안에서 auth-service 의 gRPC port 호출
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    --max-time 5 \
    http://auth-service:9005/
```

**기대 출력**: `status=000 time=N.NNNs` 또는 status code 빠르게 응답. **timeout 아님** (NetworkPolicy 통과).

(gRPC port 에 HTTP curl 이면 stream end 또는 connection close. 핵심 = TCP connection 자체 OK)

### Step 2 — monitoring → service:8005 허용

```bash
# Prometheus pod 안에서 (monitoring ns)
kubectl -n monitoring exec deploy/kube-prometheus-stack-prometheus-operator -- \
    wget -qO- --timeout=5 http://auth-service.market-dev:8005/healthz 2>&1 | head -5
```

**기대**: actuator response JSON. NetworkPolicy `prometheus-scrape-allow` 통과.

(또는 Prometheus pod 가 wget 없으면 — Prometheus targets 페이지에서 직접 scrape 확인. R-42 Part 2 참조)

---

## Part 2 — 차단 경로 검증 (Deny)

### Step 1 — product-service → auth-service:9005 차단 (핵심 demo)

평가 매니페스트의 코멘트 명시:
> 검증 절차 (R-49 task):
>   kubectl exec -n market-dev product-service-xxx -- curl auth-service:9005   → 거부 (DENY)
>   kubectl exec -n market-dev api-gateway-xxx     -- curl auth-service:9005   → 허용 (ALLOW)

```bash
# product-service 안에서 auth-service 호출 (--max-time 5 로 빠른 timeout)
kubectl -n market-dev exec deploy/product-service -c istio-proxy -- \
    curl -s --max-time 5 -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    http://auth-service:9005/ 2>&1
```

**기대**:
- `curl: (28) Operation timed out` 또는
- `status=000 time=5.NNNs` — connection 5 초 동안 응답 없음 (NetworkPolicy packet drop)

**의도**: BFF 아키텍처 — auth-service 는 api-gateway 만 호출. product-service 같은 다른 backend 가 auth-service 직접 호출 금지.

### Step 2 — order-service → product-service:9001 차단

```bash
kubectl -n market-dev exec deploy/order-service -c istio-proxy -- \
    curl -s --max-time 5 -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    http://product-service:9001/ 2>&1
```

**기대**: timeout — backend-grpc-allow-gateway 가 api-gateway 만 allow.

### Step 3 — 임의 namespace 의 pod → market-dev 차단

```bash
# default namespace 에 임시 debug pod
kubectl run nettest \
    --image=curlimages/curl:8.10.1 \
    --restart=Never \
    --rm -i \
    --command -- \
    curl -s --max-time 5 -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    http://api-gateway.market-dev.svc.cluster.local:8100/api/v1/products
```

**기대**: timeout — api-gateway-allow-ingressgateway 가 istio-system ns 만 allow.

---

## Part 3 — Allow / Deny matrix 종합 표

발표 demo 1 분 — 시각적 명확:

```bash
# 한 번에 실행. 각 줄의 결과 = ALLOW (status=xxx) 또는 DENY (timeout 5s)
echo "=== ALLOW: api-gateway → auth-service:9005 ===" ; \
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    --max-time 5 http://auth-service:9005/ ; \
echo "" ; \
echo "=== DENY: product-service → auth-service:9005 ===" ; \
kubectl -n market-dev exec deploy/product-service -c istio-proxy -- \
    curl -s -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    --max-time 5 http://auth-service:9005/ ; \
echo "" ; \
echo "=== DENY: order-service → product-service:9001 ===" ; \
kubectl -n market-dev exec deploy/order-service -c istio-proxy -- \
    curl -s -o /dev/null -w "status=%{http_code} time=%{time_total}s\n" \
    --max-time 5 http://product-service:9001/
```

**기대 출력**:
```
=== ALLOW: api-gateway → auth-service:9005 ===
status=000 time=0.0XXs    ← 즉시 응답 (gRPC port 에 HTTP 라 status=000, 단 timeout 아님)

=== DENY: product-service → auth-service:9005 ===
status=000 time=5.0NNs    ← 5초 timeout (NetworkPolicy drop)

=== DENY: order-service → product-service:9001 ===
status=000 time=5.0NNs    ← 5초 timeout
```

**판정 기준**: `time` 이 5 초 = DENY, < 5 초 = ALLOW.

---

## 평가 발표 포인트

| 항목 | 시연 |
|---|---|
| default-deny 정책 | Part 2 Step 3 — 임의 namespace 의 pod 가 차단됨 |
| 의도 통신 명시 allow | Part 1 — istio-ingressgateway / api-gateway / monitoring 만 허용 |
| BFF 아키텍처 강제 | Part 2 Step 1 — backend-to-backend 차단. api-gateway 만 backend 호출 가능 |
| Prometheus scrape 분리 | Part 1 Step 3 — monitoring ns 만 metrics port 허용 |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| Deny 검증도 즉시 응답 (5초 미만) | NetworkPolicy 미적용 — `kubectl -n market-dev get networkpolicy` 확인 + Calico CNI 작동 확인 |
| Allow 검증도 timeout | (a) pod 의 labels mismatch — `kubectl get pod <pod> -o jsonpath='{.metadata.labels}'` 확인 (b) Istio sidecar 가 outbound 차단 (Istio outboundTrafficPolicy) |
| `kubectl exec -c istio-proxy` error | sidecar 미주입 — namespace istio-injection=enabled label 확인 |
| `nettest` pod 가 NetworkPolicy 우회 | default-deny 가 market-dev namespace 안의 pod 만 적용. 다른 ns 에서 호출은 destination 의 policy 가 차단 |
| curlimages/curl image pull fail | NAT 통한 docker.io 접근 — `kubectl -n default describe pod nettest` 의 events 확인 |

---

## 정적/논리/실행/실패 흐름 검증

| 단계 | 결과 |
|---|---|
| 정적 | 6 policy 매니페스트 표준 K8s NetworkPolicy v1. podSelector/namespaceSelector 정합 |
| 논리 | Calico CNI 가 packet level enforce. sidecar 있어도 NetworkPolicy 작동 (source IP = pod IP). default-deny → 명시 allow 누락 = 차단 |
| 실행 | kubectl exec → curl --max-time 5 → Calico iptables → drop (Deny) 또는 forward (Allow) → 5s timeout 또는 즉시 응답 |
| 실패 | (a) NetworkPolicy 미sync → 모두 ALLOW (b) sidecar 미주입 → exec fail (c) label mismatch → 의도와 다른 결과 — 모두 troubleshooting |

---

## 주니어 devops 수준

- ``kubectl exec -c istio-proxy`` + ``curl`` (표준)
- ``curl --max-time 5`` + ``%{time_total}`` 으로 timeout 측정 (표준 curl 옵션)
- ``kubectl run`` 임시 pod
- 추가 도구 없음 (`tcpdump` / `iperf` / Cilium Hubble 등 미사용 — Phase 6)

---

## Reference

- [NetworkPolicy v1 reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#networkpolicy-v1-networking-k8s-io)
- [Calico NetworkPolicy enforcement](https://docs.tigera.io/calico/latest/network-policy/)
- [market-dev NetworkPolicy 매니페스트](../../platform/60-network-policies/manifests/market-dev.yaml)
