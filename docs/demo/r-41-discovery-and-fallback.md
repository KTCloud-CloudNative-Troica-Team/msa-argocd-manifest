# R-41 (B) Demo — K8s 디스커버리 + Resilience4j Circuit Breaker Fallback

> **평가 cover**
> - 기본 (2)-1 (필수): K8s ClusterIP + DNS 디스커버리 (CoreDNS)
> - 기본 (2)-3 (필수): 장애 격리 시나리오 (api-gateway ↔ inventory-service 분리)
> - 기본 (2)-4 (선택): Resilience4j Circuit Breaker Fallback (Response Aggregate)
>
> **소요**: ~10 분

## 사전 조건

cluster up + Sync Wave 완료 후 진행. master 노드에서 실행.

```bash
# 1. cluster 6 노드 Ready
kubectl get nodes

# 2. service pod Running (api-gateway / inventory-service 둘 다)
kubectl -n market-dev get pods -l 'app.kubernetes.io/name in (api-gateway,inventory-service)'

# 3. inventory-service ClusterIP + gRPC port 9003 노출 확인 (PR #92 후 OK)
kubectl -n market-dev get svc inventory-service -o jsonpath='{.spec.ports}{"\n"}'
# 기대: [{"name":"http","port":8003,...},{"name":"grpc","port":9003,...}]
```

---

## Part 1 — K8s 디스커버리 (기본 (2)-1)

CoreDNS 가 `<service>.<namespace>.svc.cluster.local` 형식 DNS 처리. service ClusterIP 가 부여됨.

> **note**: Spring Boot app container (eclipse-temurin JRE) 에는 `curl`/`wget`/`nslookup` 미설치 가능. 그러나 **Istio sidecar (`istio-proxy`)** container 에 `curl` 포함됨. sidecar 도 pod network 공유라 NetworkPolicy 통과 (api-gateway label 매칭).

### 명령어 (master 노드에서)

```bash
# A. service ClusterIP + endpoints 확인 (cluster level)
kubectl -n market-dev get svc -o "custom-columns=NAME:.metadata.name,CLUSTERIP:.spec.clusterIP,PORTS:.spec.ports[*].port"
kubectl -n market-dev get endpoints

# B. CoreDNS pod (api-gateway 의 sidecar 통한 resolve 검증)
#    curl 의 -w 옵션으로 resolved IP 출력 → DNS 작동 증명
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -o /dev/null -w "auth-service       → %{remote_ip}:%{remote_port}\n" \
    http://auth-service:8005/healthz

kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -o /dev/null -w "inventory-service  → %{remote_ip}:%{remote_port}\n" \
    http://inventory-service:8003/healthz

kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -o /dev/null -w "product-service    → %{remote_ip}:%{remote_port}\n" \
    http://product-service:8001/healthz

# C. (선택) FQDN 으로 호출 — Istio 내부 동작 확인
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -w "\nresolved: %{remote_ip}\n" \
    http://auth-service.market-dev.svc.cluster.local:8005/healthz

# D. resolv.conf — pod 의 DNS 설정 (CoreDNS IP + search domain)
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- cat /etc/resolv.conf
```

**기대 출력**:
- A: service 마다 ClusterIP (예: 10.97.x.x) + ports + endpoint pod IP 부여
- B/C: `auth-service → 10.97.x.x:8005` 등 resolved IP — DNS + routing 둘 다 작동
- D: `nameserver 10.96.0.10` (CoreDNS) + `search market-dev.svc.cluster.local svc.cluster.local cluster.local`

**평가 발표 포인트**:
- 직접 IP 가 아닌 **service 이름** 으로 호출 — pod 이동/재시작 시에도 안정
- CoreDNS 가 `.svc.cluster.local` 도메인 처리 — Kubernetes 표준
- ClusterIP 가 pod 들에 부하 분산 (Endpoints 동적 추적)

---

## Part 2 — 장애 격리 (Resilience4j Circuit Breaker Fallback)

### 사전 — ArgoCD selfHeal 일시 비활성 (CRITICAL)

inventory-service Application 의 `syncPolicy.automated.selfHeal=true` 이라 `kubectl scale` 즉시 원복됨. demo 위해 일시 비활성:

```bash
# selfHeal + auto sync 일시 비활성
kubectl -n argocd patch application inventory-service-dev \
  --type merge \
  -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 확인 (automated 가 빈 객체 또는 null)
kubectl -n argocd get application inventory-service-dev -o jsonpath='{.spec.syncPolicy}{"\n"}'
```

> **주의**: demo 끝나면 반드시 step 6 으로 selfHeal 복원.

### Step 1 — 정상 상태 응답 (control)

```bash
# api-gateway 호출 — sidecar (istio-proxy) container 의 curl 사용
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s http://localhost:8100/api/v1/inventories
```

**기대**: `{"inventories":[...]}` — inventory-service 가 정상 응답한 데이터.

### Step 2 — inventory-service 강제 down

```bash
# replica 0 — pod 즉시 종료
kubectl -n market-dev scale deploy/inventory-service --replicas=0

# pod 종료 확인 (None 또는 Terminating)
kubectl -n market-dev get pods -l app.kubernetes.io/name=inventory-service
```

### Step 3 — Fallback 트리거 검증

inventory-service down 상태에서 api-gateway 호출:

```bash
# 6 회 반복 호출 (CB sliding-window 10 의 minimum-number-of-calls=5 충족)
for i in 1 2 3 4 5 6; do
    echo "=== call $i ==="
    kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
        curl -s http://localhost:8100/api/v1/inventories
    echo ""
done
```

**기대**: 모든 호출이 `{"inventories":[]}` (빈 list) — **Fallback 작동**.

### Step 4 — Circuit Breaker 상태 확인

api-gateway 의 actuator `/healthz` 가 CB 상태 노출 (`register-health-indicator=true`):

```bash
# /healthz 응답 안에 circuitBreakers component 포함 (register-health-indicator=true)
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s http://localhost:8100/healthz \
    | grep -oE '"circuitBreakers":\{[^}]*"inventory-service":\{[^}]*\}' \
    | head -1
```

**기대 출력 (인용)**:
```
"circuitBreakers":{"status":"CIRCUIT_OPEN","details":{"inventory-service":{"failureRate":"100.0%","state":"OPEN", ...}}}
```

또는 sliding-window minimum 미충족 시 `"state":"CLOSED"` + low failure rate.

### Step 5 — Fallback log 확인

```bash
kubectl -n market-dev logs deploy/api-gateway --tail=50 \
    | grep -i "inventory-service unavailable"
```

**기대**:
```
WARN  ... inventory-service unavailable — fallback empty list: ...connection refused...
```

InventoryQueryService.kt 의 `log.warn` 메시지가 fallback path 트리거 증거.

### Step 6 — 복원 (CRITICAL)

```bash
# inventory-service replica 복원
kubectl -n market-dev scale deploy/inventory-service --replicas=1

# ArgoCD selfHeal 복원
kubectl -n argocd patch application inventory-service-dev \
  --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'

# pod 재시작 확인
kubectl -n market-dev wait --for=condition=ready pod \
    -l app.kubernetes.io/name=inventory-service --timeout=120s

# 30s 후 CB 자동 회복 (wait-duration-in-open-state=30s + HALF_OPEN 3 회 성공 → CLOSED)
sleep 35

# 정상 응답 확인
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s http://localhost:8100/api/v1/inventories
```

**기대**: `{"inventories":[...]}` 정상 응답으로 회복.

---

## Part 3 — 추가 fallback 케이스 (선택)

단일 inventory 조회의 fallback 도 별도 sentinel 값:

```bash
# inventory-service down 상태에서:
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s http://localhost:8100/api/v1/inventories/1
```

**기대**: `{"inventory":{"id":1,"productId":"","skuCode":"UNAVAILABLE","quantity":-1}}`

`skuCode="UNAVAILABLE"` + `quantity=-1` sentinel — 클라이언트가 "재고 정보 일시 사용 불가" 인지.

---

## 평가 발표 포인트 정리

| 평가요소 | 시연 |
|---|---|
| 기본 (2)-1 K8s ClusterIP + DNS | Part 1 — nslookup + wget |
| 기본 (2)-3 장애 격리 시나리오 | Part 2 Step 1~3 — inventory down 시 api-gateway 의 다른 endpoint 영향 X |
| 기본 (2)-4 Resilience4j CB | Part 2 Step 3~5 — fallback 작동 + CB state OPEN + log |
| 심화 (1)-2 Service Mesh 협업 | Istio sidecar (auth-mTLS) + ADR-0009 책임 분담 — R-03 (B) 와 함께 |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| `connection refused` 가 fallback 안 되고 그대로 응답 실패 | gRPC port 9003 미노출 (PR #92 머지 확인) |
| Fallback 안 trigger | CB instance name mismatch — `InventoryQueryService.kt` 의 `"inventory-service"` 와 application.yml 일치 확인 |
| CB OPEN 안 됨 | sliding-window 10 의 minimum-number-of-calls=5 미충족 — 호출 횟수 늘리기 |
| scale 0 직후 pod 다시 생성 | ArgoCD selfHeal 활성 — Step 0 의 patch 확인 |
| selfHeal patch 가 작동 안 함 | ArgoCD application controller 가 처리 시간 — 10 초 대기 후 재시도 |

---

## Reference

- [InventoryQueryService.kt](https://github.com/KTCloud-CloudNative-Troica-Team/msa-api-gateway/blob/main/src/main/kotlin/dev/ktcloud/black/user/api/gateway/application/inventory/service/InventoryQueryService.kt)
- [application.yaml — Resilience4j 설정](https://github.com/KTCloud-CloudNative-Troica-Team/msa-api-gateway/blob/main/src/main/resources/application.yaml)
- [ADR-0009 — api-gateway ↔ Istio Service Mesh 협업](../adr/0009-api-gateway-istio-mesh-collaboration.md)
