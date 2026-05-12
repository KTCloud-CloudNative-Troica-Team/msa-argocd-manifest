# ADR-0001: Polyrepo 구조 + auth-service 분리 + notification 폐기

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

기존 모노레포 `msa-spring-boot`에는 `user`, `auth`, `identification` 도메인 모듈이 있었으나 다음 문제가 있었다:

- 어떤 deployable로 합칠지 명세 부재. `AuthRestController`가 `@RestController` 구현 없이 인터페이스만 존재.
- `identification` 모듈의 책임 범위가 모호 (이메일 검증? 신원 인증?).
- `notification` 모듈은 코드 0줄로 그린필드 상태. SPEC §9에는 channel 분기 / 멱등성 / DLQ 등 복잡한 설계가 명시되어 있었으나 PoC 시연 범위 대비 과한 투자.
- KT Cloud Tech UP 2기 발표 일정상 작업 범위 축소 필요.

또한 polyrepo 전환 자체의 일반 원칙 "레포 1개당 1 deployable"과 위 모호함이 충돌.

## 결정

다음 세 가지를 함께 채택한다:

1. **auth-service는 신규 레포 `msa-auth-service`로 분리**. user와 같은 레포에 두지 않는다.
2. **`identification` 모듈 폐기**. 이메일 검증 등은 user-service 안에 흡수하거나 PoC에서 제외.
3. **`msa-notification-service`는 GitHub Archive**. delete가 아닌 read-only 보존.

알림 기능 자체는 운영 관측성 도구로 대체 — **Grafana → Slack webhook**으로 일원화.

## 대안

| 옵션 | 거부 사유 |
|---|---|
| (a) user/auth/identification을 한 서비스로 통합 | 도메인 경계 모호. JWT secret 등 보안 영역이 user 도메인과 섞이는 게 위험. |
| (b) auth는 별도 분리하되 identification은 user에 흡수 (채택) | 가장 자연스러운 경계. auth = 토큰 발급/검증 + 가입/로그인. user = 프로필. |
| (c) 다른 의도 | 팀이 못 떠올린 시나리오. 회의에서 (b)로 빠르게 수렴. |
| notification을 살리고 channel 분기 구현 | 발표 일정상 작업 범위 초과. 운영 알림은 Grafana로 충분. |
| notification 레포 delete | 외부 의존(common-libs:events에 `notification_requested.proto` 존재)과 history 보존을 위해 archive 선호. |

## 결과

**긍정**:
- 9개 레포 → 8개 레포로 축소 (msa-notification-service 폐기).
- auth와 user 경계 명확. auth-service는 gRPC `AuthService.{SignUp, SignIn, CheckValidity}` 단일 책임.
- JWT secret 관리가 auth-service에만 집중 → 보안 영역 격리.
- 발표 일정 안에서 구현 가능한 범위로 정리됨.

**부정 / 후속**:
- `common-libs:events`의 `notification_requested.proto`는 폐기 안 함 (D7 일관성 — 추후 subscribe 가능성). 미사용 proto가 잠시 남음.
- 이메일 검증 등 identification 기능이 누락. 추후 user-service에 추가 필요 (Phase 6 이후).
- `client-ses` 모듈도 함께 폐기 (ADR-0002).

**관련 후속 작업**:
- D6 (notification archive 실행) — 사용자 GitHub UI 수행 완료.
- R-24 (identification 흔적 제거) — Phase 4에서 확인 완료.

## 관련

- BACKLOG: D1, D6, R-05, R-24, Q2, Q3
- SPEC: §0 결정표, §1.1 서비스 목록, §9 알림 (Grafana 경유로 갱신)
- ADR-0002 (client 라이브러리 배포 — client-ses 제거가 본 결정의 연쇄)
