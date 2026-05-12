# ADR-0004: Kafka 토픽명은 SPEC 표준

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

모노레포 코드와 TROICA_SPEC.md §9 사이에 Kafka 토픽명 불일치가 발견됨:

| 영역 | order.pending에 해당하는 토픽명 |
|---|---|
| 모노레포 code | `OrderPendingEvent` (camelCase, 도메인 이벤트 클래스명 그대로) |
| TROICA_SPEC.md §9 | `order.pending` (도메인.동작 형식) |

이 외에도 4개 이벤트 모두 같은 패턴 충돌:
- `OrderInventoryReservedEvent` vs `order.inventory-reserved`
- `OrderConfirmedEvent` vs `order.confirmed`
- `OrderCancelledEvent` vs `order.cancelled`

어떤 표준을 택할지 결정 필요.

## 결정

**SPEC 표준 (`order.pending`, `order.inventory-reserved`, `order.confirmed`, `order.cancelled`)을 채택**한다.

명명 규약:
- **소문자** + **kebab-case** (다중 단어 토픽).
- **`<도메인>.<상태/동작>`** 형식.
- 도메인 분리는 `.` (dot), 단어 분리는 `-` (hyphen).

기존 코드의 클래스명은 그대로 유지 (`OrderPendingEvent` 등) — 클래스명과 토픽명은 분리.

## 대안

| 옵션 | 거부 사유 |
|---|---|
| (a) SPEC 표준 채택 (`order.pending` 등) — 채택 | Kafka 운영 표준 + AsyncAPI 등 도구 친화적. 추후 schema registry 도입 시 자연스러움. |
| (b) 모노레포 이름 유지 (`OrderPendingEvent`) + SPEC 정정 | 클래스명 = 토픽명 같지만 운영자가 클래스 PascalCase 읽기 불편. Kafka tooling 관례에 어긋남. |

## 결과

**긍정**:
- 운영 도구 친화 (kafka-console-consumer, kafdrop, AsyncAPI 등 모든 컨벤션 일치).
- 추후 Schema Registry 도입 시 subject naming 자연스러움 (e.g. `order.pending-value`).
- 새 이벤트 추가 시 명명 규약 명확.

**부정 / 후속**:
- order-service / inventory-service 모두 producer/consumer config의 토픽 상수를 수정해야 함.
- 이미 운영 중인 환경(있다면)에서 토픽 마이그레이션 필요 — 본 PoC 단계에는 운영 데이터 없어서 무관.

**후속 작업**:
- order-service의 `OrderTerminalEventKafkaPublisher`가 eventType → topic 매핑을 표준명으로 사용.
- inventory-service consumer config의 토픽 subscribe 패턴 갱신.

## 관련

- BACKLOG: Q5
- SPEC: §9 (페이로드 스키마 + 토픽 표준)
- ADR-0003 (Kafka wire format JSON 유지) — 본 ADR과 함께 Kafka 운영 표준 형성
- ADR-0006 (order state machine 7-state) — terminal 이벤트 (confirmed/cancelled)가 새 토픽 사용
