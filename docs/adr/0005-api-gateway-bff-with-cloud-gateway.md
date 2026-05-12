# ADR-0005: api-gateway = BFF (REST→gRPC) + Spring Cloud Gateway routes 혼합

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

기존 모노레포의 `user-api-gateway` 모듈과 TROICA_SPEC.md §1.1 사이에 정체성 불일치:

- **모노레포 코드**: BFF (Backend for Frontend) 구현. REST controller가 product/order/inventory의 gRPC client를 호출해서 REST로 응답.
- **SPEC §1.1**: "Spring Cloud Gateway" — reverse proxy + routing.

두 모델은 본질적으로 다름:
- **BFF**: 프론트엔드 친화적 응답 가공 (여러 백엔드 호출 → 단일 REST 응답). 비즈니스 로직 일부 포함.
- **Spring Cloud Gateway**: route table 기반 reverse proxy + filter. 비즈니스 로직 없음.

어떤 모델로 갈지 결정 필요. 또한 부수적으로 신규 레포명도 결정: 모노레포 `user-api-gateway` → polyrepo는?

## 결정

1. **두 모델을 한 deployable에 혼합 운영**한다. (Q1 의 옵션 c).
2. **신규 레포명은 `msa-api-gateway`** (모노레포의 `user-api-gateway`에서 `user-` 접두어 제거 — 'user' 도메인 한정 의도 없음).
3. 동일 서비스 안에 두 layer:
   - **BFF layer (REST controller + gRPC client)**: 프론트 친화적 API (`/api/v1/...`).
   - **SC Gateway layer (routes)**: 백엔드 직통 / admin / 기타 path (`/admin/v1/orders/**` 등).

레포는 단일 모듈 (`msa-api-gateway`). proto는 자체 보유.

## 대안

| 옵션 | 거부 사유 |
|---|---|
| (a) BFF 보존 + SPEC 정정 | SC Gateway가 주는 가치 (route weighting, retry, circuit breaker filter 등) 포기 |
| (b) SC Gateway 리라이트 + BFF 폐기 | 기존 BFF 코드 재사용 못 함. 프론트 친화적 응답 가공 (예: order + product + inventory 정보 합쳐 단일 응답) 손실 |
| (c) 한 서비스에 혼합 (채택) | BFF의 도메인 응답 가공 + SC Gateway의 인프라 patterns 둘 다 활용 |
| (d) 두 서비스로 분리 (BFF 따로, Gateway 따로) | 하나의 외부 진입점이 좋다 — 운영 + 보안 표면 단순화 |

## 결과

**긍정**:
- 단일 외부 진입점 (`msa-api-gateway` Pod). Istio Gateway 뒤에 위치.
- BFF + SC Gateway의 장점 둘 다 활용.
- 프론트 친화적 응답 (BFF)과 단순 reverse proxy (SC Gateway routes) 양립.

**부정 / 후속**:
- **모델이 섞이면 경계 모호 위험**. 어떤 path가 BFF로 가고 어떤 path가 SC Gateway route로 가는지 라우팅 매트릭스 필요. R-23으로 등재. Phase 4 진행 시 application.yaml에 명시적으로 정리됨.
- **Spring Cloud 호환성**: SB 3.5.x에 매칭되는 Spring Cloud 버전 필요. 회의 시점에는 "맞는 게 없다"는 인상이었으나 검토 결과 **Spring Cloud 2025.0.2 (Northfields, 2026-04-02 release)**가 정확히 SB 3.5.x 공식 페어. 2025.1.0은 Boot 4.x 전용으로 다른 라인. 호환 검증 완료.

**라우팅 매트릭스 (Phase 4 결정)**:
- BFF REST: `/api/v1/users/**`, `/api/v1/products/**`, `/api/v1/orders/**`, `/api/v1/inventory/**`
- SC Gateway routes: `/admin/v1/orders/**` (order admin endpoint 직통), 기타 추가 가능
- 인증: 두 layer 모두 `JwtHeaderCheckFilter`로 통합 (auth-service gRPC `CheckValidity` 호출).

## 관련

- BACKLOG: Q1, R-04, R-23
- SPEC: §1.1 서비스 목록, §1.4 외부 진입점
- TROUBLESHOOTING: §2.2 Spring Boot 3.5.13 + Spring Cloud 2025.0.2 호환성 검증
- ADR-0001 (auth-service 분리) — JWT 검증 협력 관계
