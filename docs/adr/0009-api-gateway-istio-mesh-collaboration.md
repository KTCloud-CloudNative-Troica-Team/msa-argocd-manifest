# ADR-0009: api-gateway (BFF + SC Gateway) ↔ Istio Service Mesh 책임 분담

- **상태**: Accepted
- **결정일**: 2026-05-13
- **결정자**: Troica 팀
- **관련**: R-03, R-44, ADR-0005 (api-gateway BFF + SC Gateway 혼합)

## 컨텍스트

평가 심화 (1)-2 "API 게이트웨이와 서비스 메시의 협업 구조를 검토하고 최적 구조를 선택"
에 대응. 본 프로젝트는 두 가지를 **동시 사용**할 예정임:

- **api-gateway** (외부 진입점, ADR-0005) — WebFlux + Spring Cloud Gateway routes + BFF gRPC clients
- **Istio Service Mesh** (Phase 5, R-03) — sidecar 기반 mTLS + 트래픽 시프트 + observability

두 컴포넌트가 모두 "트래픽 제어"라는 비슷한 단어로 자기 책임을 주장할 수 있어,
**중복·간섭 방지를 위한 책임 경계 명시**가 필요함.

## 결정

다음 4개 축으로 책임을 분리함.

### 1. 트래픽 진입 (north-south)

| 영역 | 담당 | 비고 |
|---|---|---|
| 외부 → 클러스터 진입 (TLS 종료, virtual host 라우팅) | **Istio ingressgateway** | ACM 인증서 / SNI 분기 |
| 클러스터 내 외부 진입 경로 응용 라우팅 | **api-gateway** | path 별 BFF 또는 SC Gateway routes |

→ Istio가 L4-L7 진입을 받고 api-gateway 서비스 1개에만 라우팅함.
→ api-gateway가 path 분기 + 서비스별 응용 호출을 책임.

### 2. 내부 통신 (east-west)

| 영역 | 담당 |
|---|---|
| 서비스 간 mTLS, 인증서 회전 | **Istio (자동 sidecar 주입)** |
| 서비스 간 비동기 통신 (Kafka) | 서비스 코드 자체 (mesh 외부) |
| 서비스 간 동기 호출 | **현재 없음** (서비스 간 직접 gRPC 호출 0건, [MSA-services.md](../images/MSA-services.md)) |

→ inter-service 동기 호출이 없으므로 Istio의 east-west 책임 = mTLS + sidecar
observability 한정.

### 3. BFF (응답 aggregation)

| 영역 | 담당 | 근거 |
|---|---|---|
| 도메인-별 응답 조합 (예: 주문 + 상품 + 재고 합성 응답) | **api-gateway only** | 도메인 지식 필요. Istio는 응용 무관. |

### 4. 회복성 (Resilience)

| 영역 | 담당 | 근거 |
|---|---|---|
| BFF 도메인 fallback (예: 재고 다운 → 재고 없이 부분 응답) | **api-gateway (Resilience4j)** | 응용 의미 — Response Aggregate (R-41) |
| 인프라 레벨 retry / connection pool / timeout | **Istio VirtualService / DestinationRule** | 인프라 |
| 인프라 레벨 outlier ejection (Circuit Breaker) | **Istio** | sidecar가 자동 처리 |

→ Resilience4j는 **도메인 fallback 한정**으로 사용. 일반 retry/timeout/CB는 Istio.
중복 회피.

### 5. JWT 검증

| 영역 | 담당 |
|---|---|
| JWT 검증 (모든 외부 요청) | **api-gateway `JwtHeaderCheckFilter` → auth-service gRPC** |
| Service-to-service JWT 전파 | (현재 없음. inter-service 동기 호출 0) |
| mTLS (서비스 ID 검증) | **Istio** |

→ JWT는 응용 인증 (사용자 신원), Istio mTLS는 워크로드 인증 (서비스 신원). 둘은 직교
관계로 동시 작동함.

## 대안 비교

| 안 | 거부 사유 |
|---|---|
| (a) api-gateway 제거, Istio Gateway + EnvoyFilter로 BFF | EnvoyFilter는 응용 라우팅 어려움. Lua/Wasm 필터 학습 곡선 큼. |
| (b) Istio 제거, api-gateway에서 mTLS도 처리 | mTLS 코드 자체 구현 부담. observability sidecar 손실. |
| (c) **혼합 (api-gateway L7 응용 + Istio L4/7 인프라)** — **채택** | 각 도구가 잘 하는 영역 분리. 학습/유지 비용 적정. |

## 결과

**긍정**
- 응용 vs 인프라 책임 명확. 발표/문서화 쉬움.
- 평가 심화 (1)-2 "협업 구조" cover.
- Phase 5 Istio 도입 시 api-gateway 코드 변경 거의 없음 (sidecar 주입만 추가).

**부정/후속**
- Resilience4j 와 Istio 회복성 정책의 경계가 운영 중 흐려질 수 있음 → 운영 가이드
  필요 (Phase 5 + 6).
- VirtualService 와 SC Gateway routes 가 같은 path를 다루지 않도록 컨벤션
  명시 필요 (예: SC Gateway는 `/admin/v1/**`, VirtualService는 `/api/v1/**` 한정).

## 관련

- BACKLOG: R-03 (Traefik → Istio), R-44 (본 ADR 작성), R-41 (응용 fallback 코드)
- ADR-0005 (api-gateway BFF + SC Gateway 혼합)
- 평가 심화 (1)-2
