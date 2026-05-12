# ADR-0008: order.confirmed / order.cancelled 토픽 발행 유지

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

ADR-0006에서 order state machine을 7-state로 확장하면서 terminal 이벤트 (`order.confirmed`, `order.cancelled`)를 Kafka에 발행하도록 결정했다. 그러나:

- 현재 PoC 시점에 이 토픽들을 subscribe할 **consumer가 없다**.
- inventory-service는 `order.inventory-reserved` 발행 측이지 terminal event consumer 아님.
- notification-service는 폐기됨 (ADR-0001).

"consumer 없는 토픽을 발행할 가치가 있나?"의 결정 필요.

## 결정

**terminal 이벤트 발행을 유지**한다. consumer가 없어도.

### 근거

1. **Outbox 일관성**: 발행 로직이 작동 보장. 향후 consumer 추가 시 즉시 동작.
2. **추후 subscribe 가능성**:
   - analytics consumer: 주문 통계 집계.
   - alerting consumer: 비정상 cancel 비율 모니터링.
   - third-party 연동: 외부 정산 시스템 등.
3. **state machine의 의미적 완전성**: terminal state에 도달하면 외부에 알리는 게 자연스러운 책임. 발행 자체가 도메인 이벤트 패턴의 일부.

## 대안

| 옵션 | 거부 사유 |
|---|---|
| terminal event 발행 유지 (채택) | 위 근거 |
| 발행 코드 작성 + Kafka topic create 안 함 | 발행 시 unknown topic 에러. 일관성 깨짐. |
| 발행 코드 자체 작성 안 함 (consumer 생기면 그때) | future-proof하지 않음. 코드 추가 시 또 결정 부담. |

## 결과

**긍정**:
- order-service의 Outbox 발행 메커니즘 완성.
- Kafka topic 4개 (`order.pending`, `order.inventory-reserved`, `order.confirmed`, `order.cancelled`)가 SPEC §9와 일치.
- consumer 추가 시 즉시 활용 가능.

**부정 / 후속**:
- Kafka broker에 "사용처 없는" 메시지 누적 → retention policy 설정 필요 (기본 168시간 = 7일이면 충분).
- 운영자가 "이 토픽 누가 듣는지" 헷갈릴 수 있음 → Phase 5에서 AsyncAPI 문서로 명시 권장.

**관련 후속 작업**:
- D7 일관성: notification-service 폐기 + terminal events 발행 유지는 별개 결정 (notification은 운영 알림이지 도메인 이벤트 아님).
- Phase 5에서 KafkaTopic CRD (Strimzi) 4개 모두 정의.

## 관련

- BACKLOG: D7
- SPEC: §9 (4 토픽 표준)
- ADR-0001 (notification 폐기와 무관 — 운영 알림 vs 도메인 이벤트 분리)
- ADR-0006 (state machine 확장) — terminal 정의의 출처
- ADR-0004 (토픽명 표준) — 본 두 토픽 명명 규약
