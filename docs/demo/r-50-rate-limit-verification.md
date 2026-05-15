# R-50 (B) Demo — RequestRateLimiter Token Bucket 작동 검증

> **평가 cover**: 기본 (2)-5 (선택) — Rate Limit
>
> **R-50 (A)**: ``msa-api-gateway/src/main/resources/application.yaml`` 의 SC Gateway ``RequestRateLimiter`` filter (Redis Token Bucket)
>
> **소요**: ~10 분

## 사전 조건

- cluster up + api-gateway pod Running (2/2 with sidecar)
- Redis (inventory-redis) Running — SC Gateway 의 backend
- R-25 / R-33 fix 후 — ConfigMap 의 ``REDIS_HOST`` 가 정확한 Spotahome service 이름

```bash
# api-gateway pod 확인
kubectl -n market-dev get pods -l app.kubernetes.io/name=api-gateway

# Redis pod 확인
kubectl -n redis get pods -l app=redis

# api-gateway 의 Redis 연결 확인 (log)
kubectl -n market-dev logs deploy/api-gateway | grep -iE "redis|lettuce" | head -5
# 기대: "Connected to ..." 또는 connection 성공 메시지 (Lettuce reactive)
```

---

## 평가 demo 의 본질

> **"동일 사용자가 burst 호출 시 429 Too Many Requests 응답"**

### Rate Limit 설정 (application.yaml)

| 경로 | burst | replenish | 적용 위치 |
|---|---|---|---|
| `/api/v1/users/**` (SC Gateway route) | 10 | 1 req/s | default-filters |
| `/admin/v1/orders/**` (SC Gateway route) | **3** (더 빡빡) | 1 req/s | route 별 filter |
| `/api/v1/products`, `/api/v1/orders`, `/api/v1/inventories`, `/api/v1/auth` (BFF RestController) | **없음** | - | SC Gateway 안 거침 |

### Key Resolver (`RateLimitKeyResolverConfig.kt`)

```
Authorization Bearer 토큰  >  client IP  >  "anonymous"
```

즉 같은 IP 또는 같은 토큰 → 같은 bucket. **cluster 안 임시 pod 의 curl loop** = 같은 pod IP → 같은 bucket → 11 번째 호출 429.

---

## Part 1 — Default filter (burst 10) 검증

### Step 1 — `/api/v1/users/**` 빠른 burst 11+ 호출

```bash
# api-gateway label 의 임시 pod (NetworkPolicy 통과 위해 api-gateway label 부여X — Istio Gateway 경유)
# 더 단순: api-gateway 의 sidecar 안에서 직접 호출 (localhost:8100)
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- /bin/sh -c '
    for i in $(seq 1 15); do
        code=$(curl -s -o /dev/null --max-time 3 -w "%{http_code}" \
            http://localhost:8100/api/v1/users/1)
        echo "$i: $code"
    done
'
```

**기대 출력**:
```
1: 404      ← user 1 없어서 404, 단 SC Gateway forward 자체는 성공
2: 404
3: 404
...
10: 404
11: 429     ← Rate Limit 발동
12: 429
13: 429
14: 429
15: 429
```

(첫 1~10 호출 가 200 또는 404 — backend (user-service) 응답에 따라 다름. 핵심 = 11+ 회 = 429)

### Step 2 — Rate limit 회복 (replenish)

```bash
# 5 초 대기 후 다시 5 회 호출 — token 5 개 회복 (1 token/s × 5s)
sleep 5
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- /bin/sh -c '
    for i in $(seq 1 5); do
        code=$(curl -s -o /dev/null --max-time 3 -w "%{http_code}" \
            http://localhost:8100/api/v1/users/1)
        echo "$i: $code"
    done
'
```

**기대**: 5 회 모두 non-429 (token 5 개 회복).

---

## Part 2 — Order admin route (burst 3, 더 빡빡) 검증

```bash
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- /bin/sh -c '
    for i in $(seq 1 8); do
        code=$(curl -s -o /dev/null --max-time 3 -w "%{http_code}" \
            http://localhost:8100/admin/v1/orders/1)
        echo "$i: $code"
    done
'
```

**기대 출력**:
```
1: 404
2: 404
3: 404
4: 429    ← burst 3 → 4 번째부터 429 (default-filters 의 burst 10 보다 더 빡빡)
5: 429
...
```

**의도**: 관리자 endpoint 는 일반 사용자 endpoint 보다 보호 강화.

---

## Part 3 — BFF path 는 Rate Limit 영향 없음 검증

```bash
# /api/v1/products 는 BFF RestController. SC Gateway filter 안 거침
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- /bin/sh -c '
    for i in $(seq 1 15); do
        code=$(curl -s -o /dev/null --max-time 3 -w "%{http_code}" \
            http://localhost:8100/api/v1/products)
        echo "$i: $code"
    done
'
```

**기대**: 15 회 모두 200 (Rate Limit 없음). `/api/v1/products` 는 BFF 가 직접 처리.

---

## Part 4 — Rate Limit Header 확인 (선택)

SC Gateway 가 응답에 `X-RateLimit-*` header 부착:

```bash
# verbose mode 로 header 까지 출력
kubectl -n market-dev exec deploy/api-gateway -c istio-proxy -- \
    curl -s -i --max-time 3 http://localhost:8100/api/v1/users/1 2>&1 | head -15
```

**기대 header**:
```
HTTP/1.1 404 Not Found
X-RateLimit-Remaining: 9
X-RateLimit-Burst-Capacity: 10
X-RateLimit-Replenish-Rate: 1
X-RateLimit-Requested-Tokens: 1
```

**확인 포인트**:
- `Remaining` 감소 → bucket 소비 확인
- 11 번째 응답 = 429 + `Remaining: 0`

---

## Part 5 — Redis backend 작동 확인 (선택)

```bash
# Spotahome 의 redis pod 발견 (정확한 service/pod 이름은 cluster 의존)
kubectl -n redis get pods
# 기대: rfr-inventory-redis-X 또는 inventory-redis-redis-X 같은 패턴

# 첫 inventory-redis 의 redis pod 으로 직접 KEYS 조회
REDIS_POD=$(kubectl -n redis get pod -l app=redis,redisfailover_name=inventory-redis \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -z "$REDIS_POD" ]; then
    echo "label 매칭 실패 — kubectl -n redis get pods 직접 확인 후 변수 수동 설정"
    # Spotahome label 패턴이 다르면 직접 pod 이름 사용:
    # REDIS_POD=<실제 pod 이름>
else
    kubectl -n redis exec $REDIS_POD -- redis-cli KEYS "request_rate_limiter*" 2>&1 | head -10
fi
```

**기대**: 적어도 1 개 key — `request_rate_limiter.{<IP>}.tokens` 등 (Part 1/2 호출 후 생성됨).

---

## 평가 발표 포인트

| 항목 | 시연 |
|---|---|
| Token Bucket 알고리즘 | Part 1 — 11 번째 429 + 5 초 후 회복 (replenish) |
| Route 별 차등 제한 | Part 2 — admin endpoint burst 3 (더 빡빡) |
| BFF vs SC Gateway 분기 | Part 3 — BFF path 는 Rate Limit 없음 |
| Redis Token Bucket backend | Part 5 — Redis KEYS 확인 |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| 모든 호출이 429 | Redis 연결 실패 — SC Gateway 가 "deny on error" 모드. ``kubectl logs deploy/api-gateway`` 의 Lettuce error 확인 |
| 어떤 호출도 429 안 나옴 | (a) RequestRateLimiter filter 미적용 — application.yaml 의 default-filters 확인 (b) Redis Token Bucket 미작동 — Redis 연결 + REDIS_HOST 값 확인 |
| Part 3 의 BFF path 도 429 발생 | application.yaml 의 default-filters 가 SC Gateway routes 만 적용되는지 확인. BFF 의 RestController 는 별도 filter chain |
| Redis 연결 fail | ConfigMap 의 REDIS_HOST 가 Spotahome 의 정확한 service 이름인지 — ``kubectl -n redis get svc`` 으로 확인 후 ConfigMap 수정 PR |
| X-RateLimit-* header 미부착 | SC Gateway 의 ``deny-empty-key`` 또는 ``add-headers-only-on-allow`` 설정 — application.yaml 의 spring.cloud.gateway.filter.request-rate-limiter 항목 확인 |

---

## 정적/논리/실행/실패 흐름

| 단계 | 결과 |
|---|---|
| 정적 | application.yaml RequestRateLimiter filter + userKeyResolver Bean 정합 ✅ |
| 논리 | Token Bucket: 11 회 호출 (1 초 안) → 10 token 소진 후 11 번째 429. replenish 1/s 라 1 초 후 1 token 회복 |
| 실행 | kubectl exec → curl loop → SC Gateway → KeyResolver (pod IP) → Redis Token Bucket → ≤10: forward / >10: 429 응답 |
| 실패 | Redis 연결 fail / filter 미적용 / 다른 path 호출 — 모두 troubleshooting |

---

## 주니어 devops 수준

- ``kubectl exec`` + ``curl`` loop (표준 shell)
- ``%{http_code}`` 으로 상태 코드 출력
- HTTP header 검사 (``curl -i``)
- ``redis-cli KEYS`` (Redis 표준)
- 추가 도구 없음 (k6 / hey / wrk / Locust 등 미사용 — 발표 demo 단순화)

> **선택**: k6 로 더 정확한 burst 측정 가능. ``kubectl run k6 --image=grafana/k6 -- run ...`` 패턴. 단 PoC demo 는 curl loop 가 충분.

---

## Reference

- [Spring Cloud Gateway RequestRateLimiter](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/gatewayfilter-factories/requestratelimiter-factory.html)
- [api-gateway application.yaml](https://github.com/KTCloud-CloudNative-Troica-Team/msa-api-gateway/blob/main/src/main/resources/application.yaml)
- [RateLimitKeyResolverConfig.kt](https://github.com/KTCloud-CloudNative-Troica-Team/msa-api-gateway/blob/main/src/main/kotlin/dev/ktcloud/black/user/api/gateway/adapter/presentation/web/configuration/RateLimitKeyResolverConfig.kt)
