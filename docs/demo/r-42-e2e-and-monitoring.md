# R-42 (B) Demo — Newman E2E + Prometheus/Grafana 실 데이터

> **평가 cover**
> - 기본 (3)-2 (필수): Postman E2E (서비스 연계)
> - 기본 (3)-3 (필수): Prometheus + Grafana
> - 기본 (3)-5 (선택): Newman CLI in CI
> - 기본 (3)-6 (선택): Kafka consumer lag (Grafana panel)
>
> **소요**: ~15 분

## 사전 조건

cluster up + Sync Wave 완료 + R-25/R-33/R-03 (B) 통과.

```bash
# 1. service pod Running (6 service, sidecar 2/2)
kubectl -n market-dev get pods

# 2. monitoring stack Running
kubectl -n monitoring get pods | grep -E "prometheus|grafana|alertmanager"

# 3. ServiceMonitor 등록 (PR 머지 후, 본 작업)
kubectl -n market-dev get servicemonitor

# 4. Grafana dashboard ConfigMap 등록
kubectl -n monitoring get cm troica-services-dashboard
```

---

## Part 1 — Postman/Newman E2E (기본 (3)-2 + (3)-5)

### 설계 결정 — cluster 안 Newman pod 실행

GitHub Actions runner 가 cluster 내부 DNS (`api-gateway.market-dev.svc.cluster.local`) 도달 못 함. NodePort 30080/30443 외부 차단 (SG). **cluster 안 Newman pod** 가 가장 안정적.

baseUrl = `http://istio-ingressgateway.istio-system.svc.cluster.local` — Istio Gateway 경유. VirtualService 의 `/api/v1/**` route → api-gateway:8100 forward.

### Step 1 — Newman pod 실행

```bash
# Newman pod — cluster 안 (market-dev), istio-ingressgateway 통해 호출
kubectl run newman -n market-dev \
    --image=postman/newman:6.2.1 \
    --restart=Never \
    --rm -i \
    --command -- newman run \
    https://raw.githubusercontent.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest/main/tests/e2e/troica-market-e2e.postman_collection.json \
    --env-var "baseUrl=http://istio-ingressgateway.istio-system.svc.cluster.local" \
    --reporters cli \
    --bail
```

**기대 출력**: 7 step 모두 `✓` (또는 일부 skipped).

```
→ signup
  ✓ status code is 200 or 201
  ✓ response has user id
→ signin
  ✓ access token received
→ products
  ✓ product list returned
→ inventories
  ✓ inventory list returned
→ order create
  ✓ order created
→ order fetch
  ✓ order matches
→ JWT check
  ✓ token valid
```

### Step 2 — 실패 시 진단

```bash
# api-gateway log 확인
kubectl -n market-dev logs deploy/api-gateway --tail=100

# 각 backend service log
kubectl -n market-dev logs deploy/auth-service --tail=30
kubectl -n market-dev logs deploy/order-service --tail=30

# Istio Gateway 의 access log
kubectl -n istio-system logs deploy/istio-ingressgateway --tail=30 | tail
```

---

## Part 2 — Prometheus + Grafana (기본 (3)-3)

### Step 1 — ServiceMonitor 등록 확인

```bash
# market-dev / market-prod 의 ServiceMonitor 6 + 6 = 12 (각 service)
kubectl get servicemonitor -A | grep -E "api-gateway|auth-service|user-service|product-service|order-service|inventory-service"
```

**기대**: 12 ServiceMonitor (env 별 6) 등록.

### Step 2 — Prometheus targets 확인

```bash
# Prometheus pod 안에서 active targets 조회
# kubectl port-forward 후 localhost:9090 접근하거나 직접 query
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 &
sleep 3

# Active targets 조회 (별도 터미널 또는 background)
curl -s http://localhost:9090/api/v1/targets | grep -oE '"health":"[a-z]+","scrapeUrl":"[^"]*"' | head -20

# 또는 Prometheus UI 직접 접근 (브라우저 또는 ssh tunnel)
# http://localhost:9090/targets — UP/DOWN 상태 확인
```

**기대**:
- 모든 service 의 endpoint 가 `"health":"up"`
- scrapeUrl 이 `http://<podIP>:8005/prometheus` 등 (path 일치)

### Step 3 — application 라벨 부착 확인 (ServiceMonitor relabeling 검증)

```bash
# 1 개 service 의 metric query
curl -s 'http://localhost:9090/api/v1/query?query=http_server_requests_seconds_count' \
    | grep -oE '"application":"[^"]+"' | sort -u
```

**기대**: `"application":"auth-service"`, `"application":"api-gateway"`, ... 등 6 service label 부착.

만약 빈 결과 — ServiceMonitor relabeling fail. Prometheus log 확인.

### Step 4 — Grafana 접근

```bash
# Grafana admin password (ExternalSecret 으로 sync 된 secret)
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d ; echo

# Grafana port-forward
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &
```

브라우저: http://localhost:3000
- user: admin
- password: (위 출력)

### Step 5 — Grafana 6 Panel 확인

좌측 메뉴 **Dashboards → Troica → 서비스 SLI 대시보드** (uid: `troica-services`)

| Panel | 확인 |
|---|---|
| 1. 요청 처리량 (req/s) | E2E 호출 후 6 service 곡선 보임 |
| 2. 에러율 (5xx) | 정상 ~0% (5% 초과 시 red) |
| 3. P99 응답 시간 | 1s 이하 |
| 4. Kafka Consumer Group Lag | order.pending / order.inventory-reserved 의 lag 0~100 (정상) |
| 5. JVM Heap 사용률 | 30-70% (정상) |
| 6. Resilience4j CB State | inventory-service = 0 (CLOSED) |

### Step 6 — Loki / Tempo datasource 확인

Grafana 좌측 **Connections → Data sources**:
- Prometheus ✅
- Loki ✅
- Tempo ✅

Loki 의 Explore — log query:
```
{namespace="market-dev"} |= "fallback"
```

→ R-41 demo 시 발생한 `inventory-service unavailable — fallback` 메시지 보임 (있다면).

---

## Part 3 — workflow_dispatch trigger (선택)

PoC 단계 — cluster 안 Newman pod 가 primary path. workflow_dispatch 는 사용자 IP 가 SG ingress 에 허용된 후 사용 가능 (Phase 6 (B) 운영 전환).

```bash
# GitHub Actions UI: msa-argocd-manifest → Actions → "E2E (Newman)" → Run workflow
# baseUrl: <외부 endpoint — bastion ssh tunnel 또는 NLB IP>
```

또는 GitHub CLI:
```bash
gh workflow run e2e-newman.yml \
    -R KTCloud-CloudNative-Troica-Team/msa-argocd-manifest \
    -f baseUrl="http://<external-ip>:30080"
```

---

## 평가 발표 포인트 정리

| 평가요소 | 시연 |
|---|---|
| 기본 (3)-2 Postman E2E | Part 1 — Newman cli 7 step 통과 |
| 기본 (3)-3 Prometheus + Grafana | Part 2 — targets up + 6 panel 실 데이터 |
| 기본 (3)-5 Newman CLI in CI | `.github/workflows/e2e-newman.yml` 매니페스트 + workflow_dispatch UI |
| 기본 (3)-6 Kafka consumer lag | Part 2 Step 5 — Panel 4 |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| Newman 의 signup 401/403 | api-gateway 또는 auth-service down — `kubectl get pods` 확인 |
| Newman 의 step 통과하나 grafana panel 데이터 0 | ServiceMonitor relabeling fail — Part 2 Step 3 확인. application label 부착 여부. |
| Prometheus targets 의 service endpoint `down` | (a) PA portLevelMtls 미적용 (PR #93 머지) (b) /prometheus path 미노출 (application.yml exposure.include 확인) |
| Grafana 6 panel 의 그래프 X | Prometheus query 의 metric 이름 mismatch — Prometheus UI 의 Query browser 에서 `http_server_requests_seconds_count{application=~".+"}` 직접 query |
| Kafka lag panel 빈 결과 | Strimzi kafkaExporter pod 가 ready 안 됨 — `kubectl -n kafka get pods` 확인 |
| CB state panel 빈 결과 | api-gateway 의 resilience4j actuator binding — `/healthz` 의 components.circuitBreakers 부분 확인 |

---

## Reference

- [Postman Collection](https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest/blob/main/tests/e2e/troica-market-e2e.postman_collection.json)
- [Newman workflow](https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest/blob/main/.github/workflows/e2e-newman.yml)
- [Grafana Dashboard ConfigMap](../../platform/30-kube-prometheus-stack/dashboards/troica-services.yaml)
- [Spring Boot Micrometer common tags](https://docs.spring.io/spring-boot/reference/actuator/metrics.html#actuator.metrics.customizing.common-tags)
